# Kanairy on your VPS (easy setup)

The VPS runs MT5, the Kanairy EA and this server. Your phone app only controls it.

    Phone app  --(signed requests, port 8766)-->  Kanairy server  <--(127.0.0.1:80)-->  KanairyEA in MT5

## Setup, three steps
1. Install **MetaTrader 5** on the VPS and open it once (so it creates its data folder). Keep the `vps` and `mt5` folders side by side.
2. Double-click **`start-kanairy.bat`** in the `vps` folder. It does the rest:
   - installs Node.js if missing (if it does, close the window and double-click again),
   - opens port 8766 in Windows Firewall (tap Yes),
   - finds MT5, copies `KanairyEA.mq5` in and compiles it,
   - prints your **Phone address** and **App key**.
3. In your VPS provider's website, open **TCP port 8766** in its firewall (security group). Windows cannot do this part for you.

Then on the phone: Kanairy, Setup, enter the address and key, tap Connect. Account, enter your broker login (demo first), tap Launch MT5.

## Keep it running (set once)
- Right-click `autostart-kanairy.bat`, choose Run as administrator. Kanairy server then starts every time you sign in to the VPS.
- The server now starts MT5 by itself with your saved login (set `"autoLaunch": false` in `config.json` to turn that off), checks every 15 seconds that MT5 is alive, and restarts it if the EA does not connect within 2 minutes.
- It also writes the WebRequest permission into MT5's own settings every launch, so you never tick it by hand.
- After a full reboot, reconnect with Remote Desktop once so the Windows desktop exists. For no-touch reboots, turn on auto-logon (Win+R, `netplwiz`).
- Allocate an Elastic IP in EC2 and attach it to the instance so the phone address never changes.
- The app's EA row now tells you why it is not connected (MT5 not running, wrong token, EA silent).

## If something does not work
- **First, double-click `fix-everything.bat`.** It checks every link (MT5, EA, token, WebRequest permission, Windows firewall, server, AWS port, clock), repairs what it can, and prints exactly what is left. Report is saved in `data\doctor-report.txt`.
- `MT5 not found`: install MetaTrader 5, open it once, run the bat again. Or set `mt5Path` and `mt5DataDir` in `config.json` by hand.
- EA did not compile: open MT5, press F4, open KanairyEA, press F7, and read the errors.
- Phone cannot connect: open `http://YOUR_VPS_IP:8766/api/status` in the phone browser. `{"error":"unauthorized"}` means the server is reachable. If the page never loads, port 8766 is closed in the provider firewall.

## Linux
`node server.js` works the same. MT5 needs Wine there. Set `mt5Path` to the Wine path of `terminal64.exe`.

## Security
- Every app request is signed with your App key and expires in 60 seconds, so nobody can place or replay trades without the key. The key never travels over the network.
- Traffic itself is plain HTTP. Balances and trades are visible to someone on your network path. For privacy put the server behind HTTPS (Caddy or Cloudflare Tunnel) or a VPN such as Tailscale, and use that address in the app.
- The EA bridge (port 80) listens on the VPS itself only. Do not open it in the AWS security group. MetaTrader's WebRequest can only reach the standard web ports, which is why it is 80. If another program uses port 80 (IIS, for example), stop that program.
- Your broker password is stored encrypted in `data\secrets.json` and written to MT5's start file only while MT5 starts (removed after 90 seconds).
- The EA refuses real accounts unless `allowLive` is `true` in `config.json`. Use a demo account first.

## Import your own EA (.mq5)
In the app, Strategy tab, **Import an EA**. Pick a `.mq5` source (this server compiles it with MetaEditor) or a compiled `.ex5` (copied into MT5 as it is) and it runs inside MT5.
- An `.ex5` cannot be inspected by Kanairy, so only use ones you trust and are licensed for. EAs that need a DLL are refused (Kanairy keeps DLLs off). EAs that `#include` your own files will fail to compile; the error shows in the app.
- Optional: add the EA's `.set` settings file, choose the chart timeframe, then tap **Run it on my VPS**. The symbol is the one set in Account.
- An imported EA trades **by itself**. Kanairy's guard rails, the live-account lock (`allowLive`) and the Live crown do not apply to it. Use a demo account first.
- The Live screen's stop button shuts MT5 down so the EA cannot trade. Open trades stay open with their broker stops.
- **Back to my XML strategy** switches back to KanairyEA. Imported files live in MT5's `MQL5\Experts\Imported` folder; compile output is in `data\ea-compile.log`.
- Only import EAs you own or are licensed to use.
