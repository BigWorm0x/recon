### Recon Scoping Writeup

Reconnaissance (recon) scoping is the foundational phase of any vulnerability assessment or penetration testing engagement. It involves gathering detailed information about the target systems, networks, and infrastructure to define the scope of the assessment and identify potential attack vectors. Below is a detailed breakdown of the recon scoping process, including tools and techniques used at each step.

---

### **Step 1: Subdomain Enumeration**
Subdomain enumeration is the process of discovering subdomains associated with the target domain. This can be done using both passive and active techniques.

#### **Tools and Commands:**
1. **Passive Subdomain Enumeration with Subfinder:**
   ```bash
   subfinder -d gcash.com -o ~/recon/targets/gcash.com/subdomains/subfinder.txt
   ```
   - Subfinder is a fast and efficient tool for passive subdomain discovery.

2. **Certificate Transparency Logs with Assetfinder:**
   ```bash
   assetfinder --subs-only gcash.com >> ~/recon/targets/gcash.com/subdomains/assetfinder.txt
   ```
   - Assetfinder leverages certificate transparency logs to find subdomains.

3. **Dynamic Subdomain Enumeration with Alterx and DNSx:**
   ```bash
   echo gcash.com | alterx -enrich | dnsx > ~/recon/targets/gcash.com/subdomains/alterx-dynamic.txt
   ```
   - Alterx generates permutations of subdomains, and DNSx resolves them.

4. **Permutation-Based Subdomain Enumeration:**
   ```bash
   echo gcash.com | alterx -pp 'word=subdomains-top1million-50000.txt' | dnsx > ~/recon/targets/gcash.com/subdomains/alterx-permutation.txt
   ```
   - This technique uses a wordlist to generate potential subdomains.

5. **ASN Mapping with Asnmap and DNSx:**
   ```bash
   asnmap -d gcash.com | dnsx -silent -resp-only -ptr > ~/recon/targets/gcash.com/subdomains/dnsx.txt
   ```
   - ASN mapping helps identify subdomains associated with the target's autonomous system number (ASN).

6. **Virtual Host Enumeration with FFUF:**
   ```bash
   cat subdomains-top1million-50000.txt | ffuf -w -:FUZZ -u http://gcash.com/ -H 'Host:FUZZ.gcash.com' -ac
   ```
   - FFUF is used to discover virtual hosts by fuzzing the `Host` header.

---

### **Step 2: Merging and Deduplicating Subdomains**
After enumerating subdomains using multiple tools, the results often contain duplicates. These need to be merged and deduplicated for efficiency.

#### **Command:**
```bash
cat ~/recon/targets/gcash.com/subdomains/*.txt | anew ~/recon/targets/gcash.com/subdomains/subdomains.txt
```
- `anew` is used to merge files and remove duplicate entries.

---

### **Step 3: Filtering Live Subdomains**
Once the subdomains are consolidated, the next step is to identify which ones are live (i.e., hosting active HTTP/HTTPS services).

#### **Command:**
```bash
cat ~/recon/targets/gcash.com/subdomains/subdomains.txt | httpx -o ~/recon/targets/gcash.com/subdomains/httpx.txt
```
- `httpx` is a tool for probing live servers and filtering out non-responsive subdomains.

---

### **Step 4: Information Gathering**
With the list of live subdomains, the next step is to gather information about the technologies used by the target.

#### **Technique:**
- Use `httpx` with Wappalyzer integration to identify technologies (e.g., CMS, frameworks, libraries).
- Example:
  ```bash
  cat ~/recon/targets/gcash.com/subdomains/httpx.txt | httpx -tech-detect -o ~/recon/targets/gcash.com/tech.txt
  ```

---

### **Step 5: Vulnerability Scanning with Nuclei**
Nuclei is a powerful tool for scanning vulnerabilities across the target infrastructure.

#### **Command:**
```bash
cat ~/recon/targets/gcash.com/subdomains/httpx.txt | nuclei -config ~/nuclei-templates/config/custom.yml
```
- Nuclei uses predefined templates to scan for vulnerabilities.

---

### **Step 6: AWS S3 Bucket Enumeration**
If the target uses AWS, it’s important to check for misconfigured S3 buckets.

#### **Commands:**
1. **Filter S3 Buckets:**
   ```bash
   cat ~/recon/targets/gcash.com/subdomains/httpx.txt | \
   nuclei -t technologies/aws/aws-bucket-service.yaml | \
   awk -F’://’ ‘{print $2}’ | \
   awk -F’/’ ‘{print $1}’ > ~/recon/targets/gcash.com/aws/buckets.txt
   ```

2. **Check for Open Buckets:**
   ```bash
   cat ~/recon/targets/gcash.com/aws/buckets.txt | \
   xargs -I {} sh -c ‘if aws s3 ls “s3://{}” — no-sign-request 2>/dev/null; \
   then echo “{}” >> ~/recon/targets/gcash.com/aws/open_buckets.txt; fi’
   ```

---

### **Step 7: Passive Open Port Scanning**
Port scanning helps identify open ports and services running on the target.

#### **Tool:**
- **Naabu** is a fast and reliable port scanner.
  ```bash
  cat ~/recon/targets/gcash.com/subdomains/subdomains.txt | naabu --passive
  ```
  - Naabu can perform passive scans to avoid detection.

