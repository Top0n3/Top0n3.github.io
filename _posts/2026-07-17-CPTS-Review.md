---
title: "Hack The Box CPTS Review: Path, Exam, Preparation, and Personal Verdict"
date: 2026-07-17 10:00:00 +0000
categories: [Certification, Hack The Box]
tags: [Pentesting, CPTS, Exam, Hack The Box, Review]
description: "A practical, no-spoiler review of the Hack The Box CPTS path, exam, preparation strategy, and value."
toc: true
---

Hello pentesters!

# My Hack The Box CPTS Review: A No-Spoiler Retrospective

The Hack The Box Certified Penetration Testing Specialist (CPTS) is one of the most demanding practical certifications available to aspiring penetration testers. It is not a multiple-choice examination and it is not simply a collection of machines to compromise. It evaluates whether you can conduct a realistic penetration test, follow an organised methodology, chain vulnerabilities, document your evidence, and communicate the result in a professional report.

This review is based on my CPTS presentation for the HTB Meetup Benin community. It covers the certification, the official path, the exam, preparation strategy, useful resources, comparisons with PNPT and CWES, and my personal opinion. It deliberately contains no exam spoilers.

> **At a glance:** CPTS is a ten-day, hands-on penetration-testing assessment built around the 28-module Penetration Tester path. Your technical work and your professional report both matter.
{: .prompt-info }

## CPTS in 60 seconds

| Exam window | Official path | Path estimate | Voucher | Attempts |
|:---:|:---:|:---:|:---:|:---:|
| **10 days** | **28 modules** | **342 hours** | **≈ $210*** | **2** |

\* Before VAT; prices and offers may vary by country and change over time.

The certification consists of:

- **Ten days** to test the lab and submit the report
- **Twenty-eight modules** in the Penetration Tester Job Role Path
- Approximately **342 hours** estimated by Hack The Box for the path
- An exam voucher priced at approximately **$210 before VAT**, depending on location and taxes
- **Two attempts** included with the voucher

The central idea is simple: CPTS measures your ability to perform a realistic penetration test involving reconnaissance, web and internal exploitation, privilege escalation, Active Directory, pivoting, vulnerability chaining, and professional reporting.

It is therefore much more than “pwn some boxes”. It is a certification of methodology, persistence, controlled exploitation, documentation, and client-oriented communication.

> **Core idea:** CPTS evaluates whether you can take an engagement from scope and reconnaissance to exploitation, impact, remediation, and a report another professional can follow.
{: .prompt-tip }

## Why CPTS is different

CPTS is a job-role certification rather than a conventional quiz. There is no multiple-choice test. Instead, the candidate works in a dedicated lab environment and must demonstrate technical progress before submitting a report in English using the official format.

The certification rewards a complete engagement process:

- structured reconnaissance and iterative enumeration;
- hypotheses that are tested and validated;
- exploitation with clear proof and impact;
- scope management and careful documentation;
- clean screenshots and reproducible steps;
- actionable remediation advice;
- a professional report that a client can understand and use.

The important skill is being able to follow an attack chain from the first discovery to the final impact, and then explain that chain clearly.

## Who is it for?

CPTS is especially suitable for junior or intermediate penetration testers who want to build a structured methodology. It can also be valuable for bug bounty hunters who want to expand beyond web applications into infrastructure and Active Directory, cyber-security students seeking a deep practical certification, and blue-team professionals who want to understand realistic attack paths.

Before starting, I recommend being comfortable with:

- TCP/IP, ports, services, DNS, HTTP/HTTPS, SMB, and Kerberos;
- Linux and Windows command lines, permissions, processes, and files;
- web testing concepts such as Burp Suite, fuzzing, authentication, SQL injection, XSS, file uploads, LFI/RFI, and basic API testing;
- Active Directory enumeration, Kerberoasting, lateral movement, and privilege escalation;
- taking organised notes, troubleshooting patiently, and writing technical reports.

You do not need to be an expert in every area, but weak fundamentals will make the path and the exam significantly more difficult.

## The official Penetration Tester path

The Penetration Tester Job Role Path is the foundation of the certification. It contains 28 modules, 495 sections, approximately 1,970 required cubes, 450 reward cubes, and 23 Skill Assessments. Hack The Box estimates the complete path at around 342 hours, although the real time depends heavily on previous experience and study pace.

The path is designed to prevent a “random box grinding” approach. It takes the learner through the fundamentals, reconnaissance, web testing, exploitation, privilege escalation, Active Directory, pivoting, and reporting before reaching the capstone-style material.

### Modules and skills covered

The modules can be viewed as a progression from the engagement scope to the final report:

1. **Methodology and fundamentals:** penetration-testing process, scope, objectives, rules of engagement, and getting started.
2. **Reconnaissance and enumeration:** Nmap, footprinting, web information gathering, vulnerability assessment, file transfers, shells, and payloads.
3. **Web exploitation:** proxies, FFUF, authentication attacks, SQL injection, SQLMap, XSS, file uploads, file inclusion, command injection, and common web applications.
4. **Services and exploitation:** password attacks, common services, Metasploit, and adapting exploits to the target.
5. **Post-exploitation:** Linux and Windows privilege escalation, credential discovery, loot, and limited persistence concepts.
6. **Internal networks and Active Directory:** AD enumeration and attacks, Kerberos, BloodHound, lateral movement, pivoting, tunnelling, and port forwarding.
7. **Reporting:** documenting findings, explaining impact, presenting evidence, and writing useful remediations.

The most important habit the path develops is iterative enumeration. Every new foothold should trigger another round of discovery: local services, credentials, routes, shares, users, and reachable networks may expose the next part of the attack path.

## What the path teaches

The value of the path is not limited to memorising commands. It teaches how to build a hypothesis from incomplete information, identify relationships between systems, adapt an exploit, and understand the business impact of a compromise.

It also develops practical network skills. Tunnels, port forwarding, proxychains, and related tools become necessary when the next target is not directly reachable. In parallel, the reporting modules teach how to turn technical activity into an executive summary, a coherent attack story, evidence, risk ratings, and actionable recommendations.

Finally, the path teaches discipline. Maintaining an activity log, versioning notes, recording failed attempts, and avoiding rabbit holes are not administrative tasks; they directly improve technical performance.

## Cost and access options

The CPTS exam voucher costs approximately **$210 before VAT** and is valid for one year. It includes two attempts. The Penetration Tester path must be completed before beginning the exam.

The path may be accessed using Academy cubes or through an Academy subscription, depending on the learner’s account and current offer. A student subscription can be a particularly affordable option. Additional practice is available through VIP+ and Pro Labs; these are useful but not mandatory. Prices and offers may change, so check Hack The Box before purchasing.

## The CPTS exam

The exam provides ten days to test a dedicated lab environment and submit a professional report. The candidate may use Pwnbox or a personal setup, and the lab may provide reset functionality. The objectives combine technical compromise, progressive attack paths, flags, evidence, and a complete report submitted through the official dashboard.

The report must be written in English and follow the HTB template. The final result depends on both technical performance and report quality. A technically successful engagement can still be weakened by missing evidence, unclear attack paths, poor remediation, or incomplete documentation.

> **No-spoiler boundary:** this review does not discuss the number of machines, specific vulnerabilities, expected techniques, or exam-specific attack paths.
{: .prompt-warning }

I will not discuss the number of machines, specific vulnerabilities, expected techniques, or any exam-specific attack path. Those details would undermine the purpose of the certification and violate the spirit of a fair assessment.

## How to use the ten days

A practical schedule might look like this:

- **Day 0 — Setup:** configure the VPN or Pwnbox, prepare note templates, create an evidence directory, and verify tools.
- **Days 1–2 — Broad reconnaissance:** map hosts, services, web applications, and initial hypotheses.
- **Days 3–4 — Initial footholds:** investigate and document web and service exploitation, credentials, shells, and early objectives.
- **Days 5–6 — Privilege escalation and pivoting:** escalate access, collect loot, map routes, and reach previously unavailable networks.
- **Days 7–8 — Active Directory and chaining:** investigate identity relationships, lateral movement, and overall impact.
- **Days 9–10 — Report hardening:** complete screenshots, verify findings, write recommendations, check consistency, and perform a final review.

This is not a rigid timetable, but it illustrates the most important rule: begin the report on the first day. A vulnerability that is not documented properly can become a finding that is effectively lost.

```text
Reconnaissance  →  Foothold  →  Privilege escalation  →  Pivot  →  AD chain  →  Report
      J1–J2           J3–J4             J5–J6             J5–J6       J7–J8       J9–J10
```

## How to prepare intelligently

I recommend eight preparation pillars:

1. **Complete the path at 100%.** Understand each module instead of simply clicking through it.
2. **Redo the Skill Assessments.** Work without solutions or unnecessary dependence on notes.
3. **Complete the CPTS Track.** Use it to close gaps and practise chaining techniques.
4. **Use the HTB Discord carefully.** Read the FAQ and ask methodological questions without requesting spoilers.
5. **Maintain clean notes.** Organise procedures by service, scenario, decision, and next action.
6. **Build operational cheat sheets.** Include complete commands, variables, purpose, interpretation, and expected output.
7. **Use targeted labs.** Practise weaknesses in AD, pivoting, privilege escalation, or internal web testing.
8. **Complete the AEN module blind.** Treat it as a final methodology and reporting test, without relying on a write-up.

The correct mindset is: **understand, then memorise, then automate**. If you can only reproduce a technique by copying a solution, you are not ready to rely on it under exam pressure.

> **Readiness test:** you should be able to redo a comparable Skill Assessment with your own notes, explain why each step works, and recover calmly when your first idea fails.
{: .prompt-tip }

## Notes and operational cheat sheets

Good notes should tell you what to test next, not merely record what you already did. For each procedure, record the context, prerequisites, command, expected output, interpretation, and next decision. Evidence should include the IP address, hostname, user, date, command, meaningful output, readable screenshot, impact, and complete exploitation path.

For example, an Active Directory cheat sheet can describe when to test Kerberoasting, the required domain and credentials, the relevant command templates, how to validate a returned hash, and what credential-reuse checks should follow. The same structure is useful for SMB, LDAP, MSSQL, WinRM, FTP, SNMP, web testing, pivoting, and Linux or Windows privilege escalation.

## Useful labs and resources

The CPTS Track, the official path, and the Skill Assessments should come first. Pro Labs and retired machines are optional reinforcements, not substitutes. Dante, Zephyr, and Offshore are examples of labs that can help practise corporate networks, Active Directory, and pivoting, depending on your level.

After each lab, extract a small number of reusable procedures and document the engagement as a mini-report. The purpose of labs is to practise methodology, not to collect flags or consume hundreds of unrelated resources.

Other useful resources include the HTB CPTS Discord, carefully selected experience reports, the Hack The Box subreddit, SysReptor for report-writing practice, and Obsidian or another structured note-taking tool. Prefer resources focused on preparation, strategy, and reporting rather than material that attempts to spoil the exam.

## Difficulty and common mistakes

CPTS is difficult because of the volume of the path, the density of the technical material, the length of the enumeration process, the ten-day schedule, fatigue, and the mandatory report. Previous experience with PNPT, solid web fundamentals, privilege-escalation practice, Active Directory, and a reliable note-taking system can help considerably.

The most common mistakes are:

- rushing through the fundamentals;
- relying entirely on LinPEAS or WinPEAS without manual verification;
- failing to map the network before pivoting;
- taking unreadable or incomplete screenshots;
- spending too long in rabbit holes without timeboxing them;
- leaving the report until the final hours;
- consuming write-ups instead of building independent problem-solving ability.

Automation is useful, but it does not replace understanding. A tool can identify a possibility; the tester must validate it, explain its impact, and document it.

<details>
<summary>Quick preparation checklist</summary>

- Finish the official path and review weak modules.
- Redo the Skill Assessments without following solutions.
- Complete the CPTS Track and AEN module without a write-up.
- Prepare service-based notes and an evidence directory.
- Practise writing a finding with evidence, impact, and remediation.
- Timebox rabbit holes and protect time for the report.

</details>

## CPTS vs PNPT

Both certifications are practical, but they emphasise different experiences.

| Criterion | CPTS | PNPT |
|---|---|---|
| Provider | Hack The Box Academy | TCM Security |
| Format | Ten-day lab, flags, and report | Five-day pentest, two-day report, and live debrief |
| Focus | Broad technical depth: web, internal networks, AD, pivoting, and reporting | Realistic network/AD engagement, OSINT, domain compromise, and client communication |
| Preparation | The Academy path is required | Training is included, without the same mandatory path model |
| Strength | Large volume of hands-on technical practice | Strong mission and client-facing realism |
| Limitation | Can feel technically dense and HTB-like | Generally less broad in web and Academy-style depth |

In my view, PNPT is particularly valuable for learning the rhythm of a client engagement and the debrief process, while CPTS offers a very deep technical learning experience and a structured progression.

## CPTS vs CWES

Choose CPTS first if you want a broad penetration-testing foundation covering networks, privilege escalation, Active Directory, pivoting, and reporting. It is also a logical choice if you already have PNPT and want to deepen your technical practice or if you are considering a future red-team path such as CAPE, CRTO, or OSEP.

Choose CWES first if your main objective is web penetration testing, bug bounty work, or modern application-security techniques. If your infrastructure and Active Directory fundamentals are already strong, specialising in web security may provide better immediate value.

My recommendation is that CWES can be a very coherent complement before CPTS: a strong web foundation makes the broader CPTS path easier, while CPTS then adds infrastructure, AD, pivoting, privilege escalation, and engagement-wide methodology.

## My personal verdict

CPTS is one of the strongest practical certifications for building a genuine intermediate penetration-testing methodology. It is long, demanding, and sometimes frustrating, but the difficulty is also what makes it valuable.

The return on investment is excellent for technical development. Market recognition will vary by country and employer, so the certification should be viewed primarily as a way to build demonstrable capability rather than as a guaranteed hiring shortcut.

My advice is straightforward: complete the path seriously, redo the assessments that exposed weaknesses, build notes that are usable under pressure, practise targeted labs, and start writing the report from the first day of the exam. Avoid excessive write-ups, avoid starting Pro Labs before fixing your fundamentals, and do not neglect web testing.

CPTS is not a shortcut. It is a structured opportunity to learn how to think, test, document, and communicate like a penetration tester.

> **Final recommendation:** take CPTS when you are ready to work methodically for ten days—not merely when you have memorised enough commands.
{: .prompt-info }

Good luck to everyone preparing for it, and stay within the authorised scope of every lab and engagement.

— *Top0n3, HTB Meetup Benin*
