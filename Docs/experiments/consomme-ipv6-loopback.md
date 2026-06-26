# Experiment: IPv6 loopback port publishing under podman + Consomme

> **Status:** throwaway experiment. All changes below are marked `TEST`/`EXPERIMENT`
> in the code and are meant to be stashed on a branch to validate the approach
> end-to-end. None of it is production-ready yet (see
> [Known gaps](#known-gaps--what-proper-looks-like)).

## TL;DR

`WSLCTests::PortMappingsConsomme` fails for IPv6: a container port published with
podman is reachable from the host on `http://127.0.0.1:<port>` but **not** on
`http://[::1]:<port>`. Docker works; podman does not. The difference is that
podman uses **netavark** (nftables DNAT) for rootful bridge port publishing, and
netavark cannot forward packets that consomme sourced/dialed using **link-local**
IPv6 addresses.

The fix has two halves — fix the **destination** consomme dials, and fix the
**source** consomme uses — plus one **manual runtime step** to give the default
podman network an IPv6 subnet at all.

> ⚠️ **Manual step you must not forget:** enable IPv6 on the podman default
> network by editing **`/etc/containers/networks/podman.json`** (see
> [Enable IPv6 on the podman default network](#enable-ipv6-on-the-podman-default-network)).
> Without an IPv6 subnet on the `podman` bridge, containers have no IPv6 address
> and `[::1]:<port>` publishing has nothing to reach — the code changes below are
> necessary but not sufficient on their own.

## Topology / background

`loopback0` is the consomme localhost-relay virtio device inside the VM:

- IPv4: `169.254.73.250/29`
- IPv6 (SLAAC from MAC `00:11:22:33:44:55`): `2001:abcd::211:22ff:fe33:4455`

Consomme advertises the prefix `2001:abcd::/64` via RA **with the on-link flag
omitted** (`NETWORK_PREFIX_BASE` in `openvmm` `…/net_consomme/consomme/src/ndp.rs`),
so the prefix is *off-link* — the guest only has `/128` host routes for it, no
connected `/64`. The loopback gateway the guest routes through is
`fe80::500:4aef:feef:2aa2`, statically mapped to MAC `00:11:22:33:44:55` so
consomme can identify loopback frames.

When the host opens `http://[::1]:<port>`, consomme NATs that connection into the
guest and dials the guest's routable loopback0 address; netavark then DNATs to the
container.

## Symptom & root cause

Captured with `tcpdump` on loopback0. Two independent bugs, both rooted in
consomme using link-local IPv6 where a *routable* address is required:

1. **Destination (which guest address consomme dials).**
   Consomme learns the guest's routable IPv6 from the **DAD Neighbor
   Solicitation** the guest emits for its loopback0 SLAAC address. DAD was
   **disabled** on loopback0, so consomme never learned the routable address and
   fell back to dialing the guest's **link-local**. netavark's DNAT rules match
   the routable published address, not the link-local one, so the SYN was never
   forwarded to the container.

2. **Source (which address consomme uses as the packet source).**
   Consomme source-NATs each host-side local address to a unique *virtual* address
   so replies route back through the arrival interface. For IPv6 it minted a
   **link-local** virtual source (`fe80::ff:fe00:NNNN:1`). A link-local source is
   interface-scoped and non-forwardable, so when the container replied, netavark /
   the guest stack had no way to route the reply back out loopback0.

(For contrast, IPv4 "link-local" `169.254.0.0/16` *is* forwardable, which is why
the IPv4 case worked and masked the problem.)

## Changes made

### openvmm (`net_consomme` → `wsldevicehost.dll`)

| File | Change |
|---|---|
| `vm/devices/net/net_consomme/consomme/src/local_addr_map.rs` | `get_or_allocate_v6` now allocates the virtual source from consomme's **routable** advertised prefix `2001:abcd::ff:fe00:NNNN:1` instead of link-local `fe80::ff:fe00:NNNN:1`. The IID `00ff:fe00:NNNN:0001` is deliberately distinct from EUI-64 (`…ff:fe…`) so it never collides with the guest's own SLAAC addresses. `test_allocate_v6` updated to expect `2001:abcd::00ff:fe00:1:1`. |

> The destination half (half 1 above) is fixed purely on the WSL side by
> re-enabling DAD — see below — because consomme already learns the routable
> address from the DAD NS once DAD is on.

### WSL

| File | Change |
|---|---|
| `src/windows/common/ConsommeNetworking.cpp` | `SetupLoopbackDevice()`: `createLoopbackDevice.flags = hns::CreateDeviceFlags::None;` (was `DisableDAD`). Re-enabling DAD lets consomme learn the guest's routable loopback0 IPv6 from the DAD Neighbor Solicitation. |
| `src/linux/init/NetworkManager.cpp` | `InitializeLoopbackConfiguration()`: added the experiment block — (a) a 5 s wait for loopback0 SLAAC + DAD to settle, (b) a static neighbor entry for the IPv6 loopback gateway, and (c) a route `2001:abcd::ff:fe00:0:0/96 via fe80::500:4aef:feef:2aa2 dev loopback0` installed into the **main** table so it is matched by IPv6's default `from all lookup main` rule (the loopback/local policy-routing tables are IPv4-only today). Added `<chrono>`/`<thread>` includes. |
| `src/windows/wslcsession/WSLCSession.cpp` | Removed the Windows-side `Sleep(5000)` from `StartPodmanSystemService()`; the wait now lives in loopback0 setup (the Linux side), tied to the actual SLAAC/DAD event rather than guessed at the Windows layer. |

#### Why the `/96` route is needed

Inbound: host `[::1]:<port>` → consomme rewrites src to `2001:abcd::ff:fe00:NNNN:1`,
dst to loopback0's routable `2001:abcd::211:…` → netavark DNATs dst to the
container.
Reply: container → conntrack reverses the DNAT so the reply is
src `2001:abcd::211:…`, dst `2001:abcd::ff:fe00:NNNN:1`. That destination is in
consomme's off-link prefix and matches **no** existing guest route, so without the
`/96` the reply is dropped. The new route sends the whole virtual range back out
loopback0 (with the gateway MAC) to consomme, which then reverse-maps the virtual
address to the real host address and delivers it.

## Enable IPv6 on the podman default network

> **This is a required manual/runtime step, not yet automated.**

The default `podman` bridge network ships IPv4-only, so containers attached to it
get no IPv6 address and published `[::1]:<port>` mappings can't reach them. Edit
**`/etc/containers/networks/podman.json`** inside the distro and give it an IPv6
subnet, e.g.:

```jsonc
{
  "name": "podman",
  "id": "...",                 // keep the existing id
  "driver": "bridge",
  "network_interface": "podman0",
  "ipv6_enabled": true,        // <-- add
  "subnets": [
    { "subnet": "10.88.0.0/16", "gateway": "10.88.0.1" },
    { "subnet": "fd00:abcd::/64", "gateway": "fd00:abcd::1" }  // <-- add an IPv6 subnet
  ],
  "ipam_options": { "driver": "host-local" }
}
```

Recreate the network instead of hand-editing if you prefer:

```bash
podman network rm podman          # only if no containers are attached
podman network create --ipv6 --subnet 10.88.0.0/16 --subnet fd00:abcd::/64 podman
```

Then restart the podman system service (or the WSLC session) so containers pick up
the IPv6 subnet.

## How to build & test

1. **openvmm change → `wsldevicehost.dll`.** `wsldevicehost.dll` is built by the
   internal `shine-oss/wsl` *DeviceHost* pipeline from `net_consomme` and published
   as the `Microsoft.WSL.DeviceHost` NuGet package. It is **not** buildable from a
   plain `openvmm` or `WSL` checkout. Build it via that pipeline and drop the
   resulting DLL into
   `packages/Microsoft.WSL.DeviceHost.<ver>/build/native/bin/<platform>/wsldevicehost.dll`
   in the WSL build. *(Open question: does that pipeline path-depend on a local
   openvmm checkout or a pinned commit? — resolve before relying on the build.)*
2. **WSL changes.** Normal Windows WSL build (`cmake . && cmake --build . -- -m`).
3. **Runtime.** Apply the [podman.json](#enable-ipv6-on-the-podman-default-network)
   step inside the distro.
4. **Test.** `bin\<platform>\<target>\test.bat /name:*PortMappingsConsomme*`
   (full build first). Or manually: publish a container port and
   `curl -v http://[::1]:<port>/`. Use `tcpdump -ni loopback0 ip6` in the distro to
   watch the SYN/DNAT/reply on the virtual address.

## Known gaps / what "proper" looks like

The experiment block in `InitializeLoopbackConfiguration` is intentionally crude:

- **Fixed 5 s sleep** → replace with a real netlink wait for "loopback0 IPv6
  address is no longer tentative" (DAD complete). Could also shorten DAD via
  optimistic DAD or a smaller `RetransTimer`.
- **Unconditional** → it runs for every loopback0 setup (mirrored *and* consomme)
  and hardcodes the `2001:abcd::` prefix. Gate it on the consomme backend and/or
  drive it from a message/flag sent by Windows, and source the prefix rather than
  hardcoding it.
- **`/96` in the main table** → fine as an experiment; revisit whether the IPv6
  loopback/local policy-routing tables (currently IPv4-only) should be wired up so
  this lives alongside the existing loopback routes.
- **podman IPv6 subnet** → automate the `/etc/containers/networks/podman.json`
  change (or the network create) as part of WSLC container setup.
- **`DisableDAD` flag** → re-enabling DAD globally on loopback0 may have other
  effects; confirm it doesn't regress the mirrored-mode loopback scenarios.

## Reference values

- Consomme prefix (RA, off-link): `2001:abcd::/64` (`NETWORK_PREFIX_BASE`, ndp.rs)
- loopback0 routable SLAAC: `2001:abcd::211:22ff:fe33:4455`
- Virtual source range (this experiment): `2001:abcd::ff:fe00:0:0/96`
  (addresses `2001:abcd::ff:fe00:NNNN:1`)
- IPv6 loopback gateway: `fe80::500:4aef:feef:2aa2` → MAC `00:11:22:33:44:55`
