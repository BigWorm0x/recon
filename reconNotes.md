## writeups
## What is recon scoping?
Reconnaissance (or recon) scoping is the initial phase of a vulnerability assessment or penetration testing engagement. It involves identifying and gathering information about the target systems, networks, and infrastructure. The primary goal is to understand the scope of the environment to be tested and to gather intelligence that might be useful during subsequent phases of the assessment
## First Step
## subdomain mapping / gathering 
# passive & active subdomain enumation using subfinder
subfinder -d gcash.com -o ~/recon/targets/gcash.com/subdomains/subfinder.txt
​
# subdomain enumeration via certificate transparency logs with assetfinder
assetfinder --subs-only gcash.com >> ~/recon/targets/gcash.com/subdomains/assetfinder.txt
​
# dynamic subdomain enumeration with alterx
echo gcash.com | alterx -enrich | dnsx > ~/recon/targets/gcash.com/subdomains/alterx-dynamic.txt

# for high chance to find subdomain
# you can generate patterns based on existing subdomains
subfinder -d tesla.com | alterx | dnsx
​
# subdomain enumaration via Permutation/Alterations with alterx
echo gcash.com | alterx -pp 'word=subdomains-top1million-50000.txt' | dnsx > ~/recon/targets/gcash.com/subdomains/alterx-permutation.txt
​
# subdomain enumeration via ASMapping
asnmap -d gcash.com | dnsx -silent -resp-only -ptr > ~/recon/targets/gcash.com/subdomains/dnsx.txt
​
# subdomain enumeration via vhost
cat subdomains-top1million-50000.txt | ffuf -w -:FUZZ -u http://gcash.com/ -H 'Host:FUZZ.gcash.com' -ac
​
## second step
after we enumerate subdomains using different tools their are some duplicates subdomains, to optimize the subdomains we will merge it using anew and will it actively remove duplicate subdomains.
# Merging subdomains from ~/recon/targets/gcash.com/subdomains/* into one file and remove duplicates
cat ~/recon/targets/gcash.com/subdomains/*.txtanew~/recon/targets/gcash.com/subdomains/subdomains.txt
 
## third step
after we merge subdomains we need to filter live subdomains using httpx to filter https/http on lists of subdomain.
# Probe for live HTTP/HTTPS servers
cat ~/recon/targets/gcash.com/subdomains/subdomains.txt | httpx -o ~/recon/targets/gcash.com
​
## Fourth step
## Information Gathering:
after we filter live subdomains we will start a information gathering using httpx to gather infomation about the domains using wappalyzer mapping techniques to identify technologies that are used to websites.
## Five step
after we gather information about tech we will start using nuclei.
cat ~/recon/targets/gcash.com/subdomains/httpx.txt | nuclei -config ~/nuclei-templates/config/custom.yml
​
## six step
AWS S3 bucket:
since we see on the httpx that they use aws, lets filter the s3 bucket using nuclei.
#filter s3 buckets and save to ~/recon/targets/gcash.com/aws/butcket.txt
cat ~/recon/targets/gcash.com/subdomains/httpx.txt | \
nuclei -t technologies/aws/aws-bucket-service.yaml | \
awk -F’://’ ‘{print $2}’ | \
awk -F’/’ ‘{print $1}’ > ~/recon/targets/gcash.com/aws/butcket.txt
​
# save open buckets as open_buckets.txt from ~/recon/targets/gcash.com/aws/butcket.txt
cat ~/recon/targets/gcash.com/aws/butcket.txt | \
xargs -I {} sh -c ‘if aws s3 ls “s3://{}” — no-sign-request 2>/dev/null; \
then echo “{}” >> ~/recon/targets/gcash.com/aws/open_buckets.txt; \fi’
Seven step 
Passive Open Port Scanning:
on open port scanning i always use naabu, it allows you to enumerate valid ports for hosts in a fast and reliable manner.
# scan all open ports except on port 80 & 443
cat ~/recon/targets/gcash.com/subdomains/subdomains.txt | naabu --passive
​
## Important Writeups
firstsight.me​
CyberIntruder - Molx32CyberIntruder | Blog
