---
# the default layout is 'page'
icon: fas fa-info-circle
order: 4
---

I'm Krzysztof (Chris), a Computer Science student at AGH University of Krakow, focused on security.

I got into security the hard way: a self-hosted service I run got hit by an adaptive, multi-stage bot attack, and I had to contain it, reconstruct what happened from a database backup and harden the setup afterwards. This blog is where I write up that kind of work.

Some of these posts are about my own mistakes. A lot of early decisions on my infrastructure look different after a year of studying security, and I'd rather document that honestly than pretend everything was right from the start. Details that could identify the services are anonymized.


## What I'm interested in

- **Web application security** - especially access control and business-logic bugs, the kind automated scanners miss and you only find by understanding how the app is supposed to work
- **Incident response and forensics** - reconstructing what happened from whatever data survived
- **OSINT** - mapping infrastructure and networks from public data
- **Automation in Python** - building my own tools when existing ones don't fit

## Background

- Java, Spring Boot, Python, Flask, SQL, Linux, Docker
- Self-hosting and securing my own services (VPS, Cloudflare, nginx/Apache)
- React Native - currently building a mobile app for my engineering thesis
- TryHackMe - regular hands-on practice, web exploitation, penetration testing and AI security

## Contact

I'm open to internship and junior roles in AppSec, SOC or penetration testing.
Reach me at <span class="copy-email" title="Click to copy">krzysztof.pieczka.mail@gmail.com<span class="copy-tip">Copied!</span></span> or on <a href="https://www.linkedin.com/in/krzysztofpieczka">LinkedIn</a>.

<script>
  document.querySelectorAll('.copy-email').forEach(function (el) {
    el.addEventListener('click', function () {
      const email = el.firstChild.textContent.trim();
      navigator.clipboard.writeText(email).then(function () {
        el.classList.add('copied');
        setTimeout(function () { el.classList.remove('copied'); }, 1200);
      });
    });
  });
</script>