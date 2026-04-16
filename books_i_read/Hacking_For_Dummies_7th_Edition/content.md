# Hacking For Dummies, 7th Edition — Table of Contents

*By Kevin Beaver, CISSP*

## Introduction ... 1

- About This Book ... 2
- Foolish Assumptions ... 2
- Icons Used in This Book ... 3
- Beyond the Book ... 4
- Where to Go from Here ... 4

---

## Part 1: Building the Foundation for Security Testing ... 5

### Chapter 1: Introduction to Vulnerability and Penetration Testing ... 7

- Straightening Out the Terminology ... 7
  - Hacker ... 8
  - Malicious user ... 9
- Recognizing How Malicious Attackers Beget Ethical Hackers ... 10
  - Vulnerability and penetration testing versus auditing ... 11
  - Policy considerations ... 11
  - Compliance and regulatory concerns ... 12
- Understanding the Need to Hack Your Own Systems ... 12
- Understanding the Dangers Your Systems Face ... 14
  - Nontechnical attacks ... 14
  - Network infrastructure attacks ... 15
  - Operating system attacks ... 15
  - Application and other specialized attacks ... 15
- Following the Security Assessment Principles ... 16
  - Working ethically ... 16
  - Respecting privacy ... 17
  - Not crashing your systems ... 17
- Using the Vulnerability and Penetration Testing Process ... 18
  - Formulating your plan ... 18
  - Selecting tools ... 21
  - Executing the plan ... 22
  - Evaluating results ... 24
  - Moving on ... 24

### Chapter 2: Cracking the Hacker Mindset ... 25

- What You're Up Against ... 25
- Who Breaks into Computer Systems ... 28
  - Hacker skill levels ... 28
  - Hacker motivations ... 30
  - Why They Do It ... 31
- Planning and Performing Attacks ... 34
- Maintaining Anonymity ... 36

### Chapter 3: Developing Your Security Testing Plan ... 37

- Establishing Your Goals ... 38
- Determining Which Systems to Test ... 40
- Creating Testing Standards ... 43
  - Timing your tests ... 43
  - Running specific tests ... 44
  - Conducting blind versus knowledge assessments ... 45
  - Picking your location ... 46
- Responding to vulnerabilities you find ... 47
- Making silly assumptions ... 47
- Selecting Security Assessment Tools ... 48

### Chapter 4: Hacking Methodology ... 49

- Setting the Stage for Testing ... 49
- Seeing What Others See ... 51
- Scanning Systems ... 52
  - Hosts ... 53
  - Open ports ... 53
- Determining What's Running on Open Ports ... 54
- Assessing Vulnerabilities ... 56
- Penetrating the System ... 58

---

## Part 2: Putting Security Testing in Motion ... 59

### Chapter 5: Information Gathering ... 61

- Gathering Public Information ... 61
  - Social media ... 62
  - Web search ... 63
  - Web crawling ... 64
  - Websites ... 64
- Mapping the Network ... 65
  - WHOIS ... 65
  - Privacy policies ... 66

### Chapter 6: Social Engineering ... 69

- Introducing Social Engineering ... 69
- Starting Your Social Engineering Tests ... 71
- Knowing Why Attackers Use Social Engineering ... 71
- Understanding the Implications ... 72
  - Building trust ... 73
  - Exploiting the relationship ... 74
- Performing Social Engineering Attacks ... 77
  - Determining a goal ... 77
  - Seeking information ... 77
- Social Engineering Countermeasures ... 82
  - Policies ... 82
  - User awareness and training ... 83

### Chapter 7: Physical Security ... 87

- Identifying Basic Physical Security Vulnerabilities ... 88
- Pinpointing Physical Vulnerabilities in Your Office ... 89
  - Building infrastructure ... 90
  - Utilities ... 91
  - Office layout and use ... 93
  - Network components and computers ... 95

### Chapter 8: Passwords ... 99

- Understanding Password Vulnerabilities ... 100
  - Organizational password vulnerabilities ... 101
  - Technical password vulnerabilities ... 101
- Cracking Passwords ... 102
  - Cracking passwords the old-fashioned way ... 103
  - Cracking passwords with high-tech tools ... 106
  - Cracking password-protected files ... 115
  - Understanding other ways to crack passwords ... 116
- General Password Cracking Countermeasures ... 121
  - Storing passwords ... 122
  - Creating password policies ... 122
  - Taking other countermeasures ... 124
- Securing Operating Systems ... 126
  - Windows ... 126
  - Linux and Unix ... 127

---

## Part 3: Hacking Network Hosts ... 129

### Chapter 9: Network Infrastructure Systems ... 131

- Understanding Network Infrastructure Vulnerabilities ... 132
- Choosing Tools ... 133
  - Scanners and analyzers ... 134
  - Vulnerability assessment ... 134
- Scanning, Poking, and Prodding the Network ... 135
  - Scanning ports ... 141
  - Scanning SNMP ... 143
  - Grabbing banners ... 144
  - Testing firewall rules ... 144
  - Analyzing network data ... 146
  - The MAC-daddy attack ... 153
  - Testing denial of service attacks ... 157
- Detecting Common Router, Switch, and Firewall Weaknesses ... 161
  - Finding unsecured interfaces ... 161
  - Uncovering issues with SSL and TLS ... 162
- Putting Up General Network Defenses ... 162

### Chapter 10: Wireless Networks ... 165

- Understanding the Implications of Wireless Network Vulnerabilities ... 166
- Choosing Your Tools ... 168
- Discovering Wireless Networks ... 168
  - Checking for worldwide recognition ... 169
  - Scanning your local airwaves ... 170
- Discovering Wireless Network Attacks and Taking Countermeasures ... 171
  - Encrypted traffic ... 173
  - Countermeasures against encrypted traffic attacks ... 177
  - Wi-Fi Protected Setup ... 179
  - Countermeasures against the WPS PIN flaw ... 181
  - Rogue wireless devices ... 181
  - Countermeasures against rogue wireless devices ... 185
  - MAC spoofing ... 185
  - Countermeasures against MAC spoofing ... 189
  - Physical security problems ... 189
  - Countermeasures against physical security problems ... 190
  - Vulnerable wireless workstations ... 191
  - Countermeasures against vulnerable wireless workstations ... 191
  - Default configuration settings ... 191
  - Countermeasures against default configuration settings exploits ... 191

### Chapter 11: Mobile Devices ... 193

- Sizing Up Mobile Vulnerabilities ... 194
- Cracking Laptop Passwords ... 194
  - Choosing your tools ... 198
- Cracking Phones and Tablets ... 199
  - Cracking iOS passwords ... 200
  - Taking countermeasures against password cracking ... 203

---

## Part 4: Hacking Operating Systems ... 205

### Chapter 12: Windows ... 207

- Introducing Windows Vulnerabilities ... 208
- Choosing Tools ... 209
  - Free Microsoft tools ... 209
  - All-in-one assessment tools ... 210
  - Task-specific tools ... 210
- Gathering Information About Your Windows Vulnerabilities ... 211
  - System scanning ... 211
  - NetBIOS ... 214
- Detecting Null Sessions ... 217
  - Mapping ... 218
  - Gleaning information ... 221
  - Countermeasures against null-session hacks ... 222
- Checking Share Permissions ... 222
  - Windows defaults ... 223
  - Testing ... 224
- Exploiting Missing Patches ... 225
  - Using Metasploit ... 225
  - Countermeasures against missing patch vulnerability exploits ... 231
- Running Authenticated Scans ... 231

### Chapter 13: Linux and macOS ... 233

- Understanding Linux Vulnerabilities ... 234
- Choosing Tools ... 235
- Gathering Information About Your System Vulnerabilities ... 235
  - System scanning ... 238
  - Countermeasures against system scanning ... 240
- Finding Unneeded and Unsecured Services ... 240
  - Searches ... 242
  - Countermeasures against attacks on unneeded services ... 242
- Securing the .rhosts and hosts.equiv Files ... 244
  - Hacks using the hosts.equiv and .rhosts files ... 244
  - Countermeasures against .rhosts and hosts.equiv file attacks ... 245
- Assessing the Security of NFS ... 247
  - NFS hacks ... 248
  - Countermeasures against NFS attacks ... 248
- Checking File Permissions ... 248
  - File permission hacks ... 248
  - Countermeasures against file permission attacks ... 248
- Finding Buffer Overflow Vulnerabilities ... 250
  - Attacks ... 250
  - Countermeasures against buffer overflow attacks ... 250
- Checking Physical Security ... 251
  - Physical security hacks ... 251
  - Countermeasures against physical security attacks ... 251
- Performing General Security Tests ... 252
- Patching ... 253
  - Distribution updates ... 254
  - Multiplatform update managers ... 255

---

## Part 5: Hacking Applications ... 257

### Chapter 14: Communication and Messaging Systems ... 259

- Introducing Messaging System Vulnerabilities ... 259
- Recognizing and Countering Email Attacks ... 260
  - Email bombs ... 261
  - Banners ... 264
  - SMTP attacks ... 266
  - General best practices for minimizing email security risks ... 275
- Understanding VoIP ... 276
  - VoIP vulnerabilities ... 277
  - Countermeasures against VoIP vulnerabilities ... 282

### Chapter 15: Web Applications and Mobile Apps ... 283

- Choosing Your Web Security Testing Tools ... 284
- Seeking Out Web Vulnerabilities ... 285
  - Directory traversal ... 285
  - Countermeasures against directory traversals ... 289
  - Input-filtering attacks ... 290
  - Countermeasures against input attacks ... 297
  - Default script attacks ... 299
  - Countermeasures against default script attacks ... 299
  - Unsecured login mechanisms ... 300
  - Countermeasures against unsecured login systems ... 303
- Performing general security scans for web application vulnerabilities ... 304
- Minimizing Web Security Risks ... 305
  - Practicing security by obscurity ... 305
  - Putting up firewalls ... 306
  - Analyzing source code ... 306
- Uncovering Mobile App Flaws ... 307

### Chapter 16: Databases and Storage Systems ... 309

- Diving Into Databases ... 309
  - Choosing tools ... 310
  - Finding databases on the network ... 310
  - Cracking database passwords ... 311
  - Scanning databases for vulnerabilities ... 312
- Following Best Practices for Minimizing Database Security Risks ... 314
- Opening Up About Storage Systems ... 314
  - Choosing tools ... 315
  - Finding storage systems on the network ... 315
  - Rooting out sensitive text in network files ... 316
- Following Best Practices for Minimizing Storage Security Risks ... 319

---

## Part 6: Security Testing Aftermath ... 321

### Chapter 17: Reporting Your Results ... 323

- Pulling the Results Together ... 323
- Prioritizing Vulnerabilities ... 325
- Creating Reports ... 327

### Chapter 18: Plugging Your Security Holes ... 329

- Turning Your Reports into Action ... 329
- Patching for Perfection ... 330
  - Patch management ... 331
  - Patch automation ... 332
- Hardening Your Systems ... 334
- Assessing Your Security Infrastructure ... 334

### Chapter 19: Managing Security Processes ... 337

- Automating the Security Assessment Process ... 337
- Monitoring Malicious Use ... 338
- Outsourcing Security Assessments ... 340
- Instilling a Security-Aware Mindset ... 342
- Keeping Up with Other Security Efforts ... 343

---

## Part 7: The Part of Tens ... 345

### Chapter 20: Ten Tips for Getting Security Buy-In ... 347

- Cultivate an Ally and a Sponsor ... 347
- Don't Be a FUDdy-Duddy ... 348
- Demonstrate That the Organization Can't Afford to Be Hacked ... 348
- Outline the General Benefits of Security Testing ... 349
- Show How Security Testing Specifically Helps the Organization ... 350
- Get Involved in the Business ... 350
- Establish Your Credibility ... 351
- Speak on Management's Level ... 351
- Show Value in Your Efforts ... 352
- Be Flexible and Adaptable ... 352

### Chapter 21: Ten Reasons Hacking Is the Only Effective Way to Test ... 353

- The Bad Guys Think Bad Thoughts, Use Good Tools, and Develop New Methods ... 353
- IT Governance and Compliance Are More Than High-Level Audits ... 354
- Vulnerability and Penetration Testing Complements Audits and Security Evaluations ... 354
- Customers and Partners Will Ask How Secure Your Systems Are ... 354
- The Law of Averages Works Against Businesses ... 355
- Security Assessments Improve Understanding of Business Threats ... 355
- If a Breach Occurs, You Have Something to Fall Back On ... 355
- In-Depth Testing Brings Out the Worst in Your Systems ... 356
- Combined Vulnerability and Penetration Testing Is What You Need ... 356
- Proper Testing Can Uncover Overlooked Weaknesses ... 356

### Chapter 22: Ten Deadly Mistakes ... 357

- Not Getting Approval ... 357
- Assuming That You Can Find All Vulnerabilities ... 358
- Assuming That You Can Eliminate All Vulnerabilities ... 358
- Performing Tests Only Once ... 359
- Thinking That You Know It All ... 359
- Running Your Tests Without Looking at Things from a Hacker's Viewpoint ... 359
- Not Testing the Right Systems ... 359
- Not Using the Right Tools ... 360
- Pounding Production Systems at the Wrong Time ... 360
- Outsourcing Testing and Not Staying Involved ... 361

---

## Appendix: Tools and Resources ... 363

## Index ... 379
