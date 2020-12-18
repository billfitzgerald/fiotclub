For Internet of Things: Checking, Looking, Understanding, Breaking -- or FIOTCLUB.

<h2>0. Goal:</h2>

Create a lightweight, standalone testing environment for ad hoc testing of websites, apps, or any service that connects to a wireless internet connection.

This system is designed to balance two core needs: relative ease of setup paired with thorough, detailed testing. Using this system, a tester has three options that are supported out of the box:

* a standard wireshark capture;
* an nmap scan of all connected devices, paired with a timed wireshark capture;
* a proxy capture that dumps encryption keys to a SSLKEYLOGFILE, paired with a wireshark scan that uses the keys to decrypt traffic captures.

Depending on the needs of the test, the tester can choose which approach is best suited for the evaluation.

<h2>1. Hardware Needed:</h2>

<ul>
<li>Laptop with wireless card and an ethernet connection (or a Raspberry Pi): NOTE: some laptops don't need the ethernet connection, but I find the ethernet connection alongside the wireless card to be a slightly cleaner setup for testing.</li>
</ul>

<h2>2. High Level Technical Overview:</h2>

This setup is pretty straightforward. The testing machine is configured to work as a wireless access point. The devices to be tested all connect to the access point, which allows the tester to capture and inspect network traffic.

The steps described deliver some targeted features designed for testing and review:

<ol>
<li>The computer functions as a wireless access point; devices connect to it, and all network traffic is routed through it.</li>
<li>MITMProxy dumps master keys into SSLKEYLOGFILE where they can be read and used by Wireshark to decrypt traffic</li>
<li>Alternately, MITMProxy doesn't need to be in play, and Wireshark can be used to capture traffic without decrypting it.</li>
<li>Because you (the tester) control both the Access Point and the devices connecting to it, you can limit the connections running through the system at any point in time; this creates a controlled and less noisy testing environment.</li>
</ol>

The steps described below cover the full setup, from installing the operating system to installing the software needed to test to choosing a VPN to help ensure a level of anonymity from vendors while testing.

<h2>3. Software Requirements and Installation Process</h2>

<h3>3A. Install Ubuntu or Raspberry Pi OS</h3>

https://help.ubuntu.com/community/Installation/FromUSBStick

https://www.raspberrypi.org/software/

Post Install:

* <code>sudo apt-get update</code><br>
* <code>sudo apt-get upgrade</code>

<h3>3B. Python:</h3>

<code>python3 -V</code><br>
<code>sudo apt install -y python3-pip</code><br>
<code>sudo apt install build-essential libssl-dev libffi-dev python-dev</code>

Optional - more information: https://www.digitalocean.com/community/tutorials/how-to-install-python-3-and-set-up-a-local-programming-environment-on-ubuntu-18-04

<h3>3C. Git:</h3>

<code>sudo apt install git</code>

Optional - more information: https://www.digitalocean.com/community/tutorials/how-to-install-git-on-ubuntu-18-04

<h3>3D. Wireshark:</h3>

Install Wireshark: <code>sudo apt install wireshark</code><br>
Configure Wireshark to not require admin rights to run: <code>sudo usermod -aG wireshark $(whoami)</code>

Optional - more information: https://linuxhint.com/install_wireshark_ubuntu/

<h3>3E. nmap (with Zenmap front end):</h3>

Install Zenmap and nmap: <code>sudo apt-get install zenmap -y</code>

<h3> 3F. Create the access point:</h3>

These instructions describe how to configure your machine to work as an access point: https://vitux.com/make-your-ubuntu-pc-a-wireless-access-point/

While the instructions are specific to Ubuntu, they also work for Raspberry Pi OS.

Once you have set up the computer to serve as an access point, put your phone in airplane mode, and then turn on your wireless. Connect to the newly created access point, and browse the web, check email, open an app, open Google Maps, and all should work normally.

<h3>3G. Install MITM Proxy and set to run in transparent mode:</h3>

Install mitmproxy: https://docs.mitmproxy.org/stable/overview-installation/

Make sure that you have net-tools installed so you can access ifconfig.

<code>sudo apt install net-tools</code>

Set up Transparent Mode: https://docs.mitmproxy.org/stable/howto-transparent/

These commands are needed to route all traffic through the proxy. 

* <code>sudo sysctl -w net.ipv4.ip_forward=1</code>
* <code>sudo sysctl -w net.ipv6.conf.all.forwarding=1</code>
* <code>sudo sysctl -w net.ipv4.conf.all.send_redirects=0</code>

Use ifconfig to get the name of your wireless card - add the name of the wireless card in WIRELESS_NAME below.

* <code>sudo iptables -t nat -A PREROUTING -i WIRELESS_NAME -p tcp --dport 80 -j REDIRECT --to-port 8080</code>
* <code>sudo iptables -t nat -A PREROUTING -i WIRELESS_NAME -p tcp --dport 443 -j REDIRECT --to-port 8080</code>
* <code>sudo ip6tables -t nat -A PREROUTING -i WIRELESS_NAME -p tcp --dport 80 -j REDIRECT --to-port 8080</code>
* <code>sudo ip6tables -t nat -A PREROUTING -i WIRELESS_NAME -p tcp --dport 443 -j REDIRECT --to-port 8080</code>

Create the SSLKEYLOGFILE in your home directory.

<code>touch /home/USERNAME/sslkeylogfile.txt</code>

Start mitmproxy and dump secrets into SSLKEYLOGFILE.

<code>SSLKEYLOGFILE="/home/USERNAME/sslkeylogfile.txt" mitmproxy --mode transparent --showhost --ssl-insecure</code>

Possible troubleshooting/refinement:

* Set up for "full transparent" https://docs.mitmproxy.org/stable/howto-transparent/#full-transparent-mode-on-linux
* Ignore some traffic: https://docs.mitmproxy.org/stable/howto-ignoredomains/

<h3>3H. Run Wireshark and decrypt traffic:</h3>

Open Wireshark, and set Wireshark to listen at the wireless adapter. 

Go to Preferences -> Protocols -> TLS, and change the (Pre)-Master-Secret log filename preference to the path set when mitmproxy was started: <code>/home/USERNAME/sslkeylogfile.txt</code>

More information: https://wiki.wireshark.org/TLS?action=show&redirect=SSL#Using_the_.28Pre.29-Master-Secret

<h3>3I. Run an automated nmap scan and wireshark (tshark) capture:</h3>

https://gist.github.com/billfitzgerald/02d90ff5dd9f580ac82b6845ed4395db

<h2>4. Usage instructions</h2

To start testing, connect all devices to be tested to the wireless access point enabled above in Step 3F.

To run a standard Wireshark scan manually, open wireshark, and set wireshark to listen on the wireless adapter. If you are unsure of the name of the wireless adapter, enter <code>ifconfig</code> to see a list of network adapters.

To run an automated nmap scan and wireshark capture of all devices connected to the network, use this testing script: https://gist.github.com/billfitzgerald/02d90ff5dd9f580ac82b6845ed4395db - this script runs an nmap scan, then a tshark capture of network traffic, and then outputs the results into a zip archive.

To set up mitmproxy and decrypt network captures using Wireshark, use the instructions in 3G and 3H, above.
