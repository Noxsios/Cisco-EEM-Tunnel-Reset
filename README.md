# Auto Tunnel Reset for GRE Tunnels on Cisco IOS Routers

BLUF: Manually checking / bumping tunnels is a pain, this configuration fixes that.

```powershell
# To download just the .txt config -
iwr "https://git.io/Jt4yl" -o "~\Downloads\Auto_Tunnel_Reset.txt"
```

## Configuration

#### Create an SLA

```tcl
ip sla 10
icmp-echo 8.8.8.8
frequency 30
exit
ip sla schedule 10 life forever start-time now
```

#### Track Status of SLA

```tcl
track 1 ip sla 10 reachability
delay down 90 up 90
exit
```

#### Event Manager Actions

```tcl
event manager applet Tunnel-Down
event timer watchdog time 30 ratelimit 45
action 001 cli command "enable"
action 002 cli command "sh track 1 | i Reachability"
action 003 regexp "Reachability is Up" "$_cli_result"
action 004 if "$_regexp_result" eq "0"
action 005 syslog msg "Echo Reply timed out; Tunnel is down"
action 006 cli command "conf t"
action 007 cli command "interface f0/0"
action 008 cli command "shut"
action 009 cli command "interface tunnel0"
action 010 cli command "shut"
action 011 cli command "do clear crypto sess"
action 012 cli command "interface f0/0"
action 013 cli command "no shut"
action 014 cli command "interface tunnel0"
action 015 cli command "no shut"
action 016 cli command "end"
action 017 syslog msg "TUNNEL HAS BEEN REINITIALIZED"
action 018 end
```

## Explanation

#### Create an SLA- Explanation

- `icmp-echo [ADDRESS] `

  - This address can be anything, I have chosen the XX for the address to be pinged, you can change this address to anything that replies to ICMP echoes that you can only ping once the tunnel is up

- `frequency 30`
  - This is the frequency of SLA 10's pings in seconds, 30 in this case
- `ip sla schedule 10 life forever start-time now`
  - Initializes SLA 10 with a lifetime of _forever_ and a start time of _now_

#### Track Status of SLA 10- Explanation

- `track 1 ip sla 10 reachability`
  - Starts track with identifier number 1 (arbitrary, can be any number 1-1000), needs to match number in `event track 1 state down`
  - Then attaches to SLA 10's reachability status
    - Status can be viewed manually at any time with `sh track 1`
- `delay down 90 up 90`
  - Delay 90s to check SLA 10 status whether up or down

#### Event Manager Actions- Explanation

- `event manager applet Tunnel-Down`
  - Creates an event manager applet called `Tunnel-Down`
- `event timer watchdog time 30 ratelimit 45`
  - Attempts to run `Tunnel-Down` every 30s, with a maximum run 1 time per 45s
- `action 001 cli command "enable"`
  - No enable password is required as the applet runs as a privileged user / whichever user created it
- `action 002 cli command "sh track 1 | i Reachability"`
  - Returns a line containing the status of track 1
- `action 003 regexp "Reachability is Up" "$_cli_result"`
  - Compares the results of the previous CLI command `"$_cli_result"` with the string `Reachability is Up`
- `action 004 if "$_regexp_result" eq "0"`
  - If statement checking the previous regex result
- `action 005 syslog msg "Echo Reply timed out; Tunnel is down"`
  - Sends the above message to the syslog, or to the screen if you are consoled in
- `action 006-0016`
  - CLI commands to reset / bump your tunnel connection, replace with whatever actions you would otherwise have to do manually
- `action 017 syslog msg "TUNNEL HAS BEEN REINITIALIZED"`
  - Sends the above message to the syslog, or to the screen if you are consoled in
- `action 018 end`
  - Closes the if statement started in action 004, actions 005 through 017 will only be run if 004 is true

---

> Credit to :
>
> > [Cisco EEM Scripting - Cisco Community](https://community.cisco.com/t5/networking-documents/cisco-eem-basic-overview-and-sample-configurations/ta-p/3148479)
> >
> > [Cisco IP SLAs - Cisco Documentation](https://learningnetwork.cisco.com/s/blogs/a0D3i000002SKN0EAO/ip-sla-fundamentals)
> >
> > [Balaji Bandi TCL Script - Cisco Forums](https://community.cisco.com/t5/network-management/tcl-script-to-ping-a-host-and-if-down-execute-some-ios-commands/td-p/3831833)
