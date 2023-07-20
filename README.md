## Better Cloudflare Workers Mailer Docs
> Last updated 19/07/2023 by Isaac Shea

Hello! Welcome to the better Cloudflare Workers Mailer docs. This document exists to provide a more comprehensive guide to the setup and usage of the Cloudflare Workers MailChannels integration. It compiles information found at numerous sources from across the internet, and attempts to be as up-to-date as possible.

## Getting started
[Source 1]
The following code is a basic overview of the code required within your Worker:
```js
async function sendEmail() { 
	let send_request = new Request("https://api.mailchannels.net/tx/v1/send", { 
		method: "POST",
		headers: { 
			"content-type": "application/json",
		},
		body: JSON.stringify({
			personalizations: [
				{ to: [{ email: "test@recipient.com", name: "Test Recipient" }], },
			],
			from: {
				email: "test@sender.com",
				name: "Test Sender",
			},
			subject: "Test email subject!",
			content: [
				{
					type: "text/plain",
					value: "Look at all this test email content. How fabulous.",
				},
			],
		}),
	});
	
	let response = await fetch(send_request);
	console.log(response.status, response.statusText);
}
```
> If the above code runs correctly, you should see `202 Accepted` in your console.

### Some notes
- **You must control the domain you send "from":** MailChannels recently made domain verification mandatory for Cloudflare Workers users, requiring the addition of a simple TXT record; see the [[#spoofing protection]] section for more information.
- **More content types than `text/plain` can be used:** my testing has included `text/html`, and more may be available.
- **Without proper DNS setup, Gmail emails will not receive emails:** presumably because of Gmail's much stricter rules on which emails it will accept, mail sent via MailChannels without correct [[#SPF setup]] will be *entirely blocked*. They will not appear in the recipient's spam folder, nor will an error be raised by the fetch request. *Mail will simply be lost to the aether.*

### Common errors
- `500 Internal Server Error` - rather counter-intuitively, this is often caused when your sender email address (`test@sender.com` in the above example) has [[#spoofing protection]] enabled, thus you cannot send an email "from" that address.

## SPF setup
[Source 2]
Although many mail providers do not require any additional setup to be able to receive emails through Workers, you may find that some providers block emails (namely Gmail). To prevent this, the sender domain (`@sender.com` in our above example) needs to have a Sender Policy Framework (SPF) record configured.

### What is SPF?
[Source 3]
> "A Sender Policy Framework (SPF) is an email authentication method that helps to identify the mail servers that are allowed to send email for a given domain."
> -Mimecast

Essentially, this needs to be setup so that providers with strict email screening (ahem, Gmail) know that your email is coming from a legitimate domain, instead of attempting to impersonate another domain.

### Updating your DNS
[Source 2]
It goes without saying that you need to know the domain that you're going to update the DNS settings for to continue. To add configure SPF record, add a `TXT` record to your domain settings with the following values (replacing `yourdomain.com` with your literal domain):
>**Location:** `yourdomain.com`
>**Type:** `TXT`
>**Value:** `v=spf1 include:relay.mailchannels.net ~all`

If your domain already has SPF DNS records configured, insert `include:relay.mailchannels.net` at the end of the existing record, before the `~all`, like so:
>**Location:** `yourdomain.com`
>**Type:** `TXT`
>**Value:** `v=spf1 include:_spf.mx.cloudflare.net include:relay.mailchannels.net ~all`



## Spoofing protection
[Source 4]
MailChannels' Domain Lockdown feature (MailChannels being the organisation Cloudflare collaborated with to provide this service) uses the DNS to prove that you control the domains you want to send from via your Worker.
The Domain Lockdown allows you to indicate a list of senders and accounts permitted to send emails from your domain. Any other accounts that attempt to send from your domain will be rejected with an error.

**Currently, three lockdown identifiers are supported:**
1. `auth` - This identifies a MailChannels customer, such as a web hosting provider, by specifying the authentication username of the customer. `auth` codes are a sequence of letters and numbers, such as `myhostingcompany`.
2. `senderid` - This identifies a specific sender identity, such as a PHP script or authenticated webmail user account. `senderid` strings specify the provider, type of identity, and the identity, all in a single string. An example of a `senderid` is `myhostingcompany|x-authuser|myusername`.
3. `cfid` - This identifies a Cloudflare Worker and is used to prevent spoofing from your domain if you wish to send email from Cloudflare Workers using the MailChannels `/send` API. An example of a `cfid` is `username.workers.dev`.

**Implementing Domain Lockdown:**
Create a DNS `TXT` record like so (replacing `yourdomain.com` with your literal domain name):
> **Value:** `_mailchannels.yourdomain.com`
> **Type:** `TXT`
> **Content:** `v=mc1 ...`

**What to include in the Content field:**
Other than `v=mc1`, other values are optional. The record can contain any number or combination of `auth`, `senderid` and `cfid` fields, including leaving the list blank. Each field must specify **only one** value.
1. To completely block MailChannels from sending any emails using your domain, leave the TXT record as:
```content
v=mc1
```
2. To lock the domain to a specific Cloudflare Workers account, use:
```content
v=mc1 cfid=username.workers.dev
```
3. To lock the domain to a single provider, use:
```content
v=mc1 auth=myhostingcompany
```
4. To lock the domain to two different providers, use:
```content
v=mc1 auth=myhostingcompany auth=anotherprovider
```
5. To lock the domain to a specific Sender-ID, use:
```content
v=mc1 senderid=myhostingcompany|x-authuser|myusername
```

### Finding your `cfid`
1. Go to https://dash.cloudflare.com/
2. Select "Workers & Pages" from the left navigation bar
3. Find your ID in the Subdomain section of your Account Details on the right

Alternatively, you can just try sending without a Domain Lockdown record present and MailChannels will generate an error message that contains your `cfid` for convenience. Cut and paste that into your `_mailchannels` DNS record.

### Finding your `auth` or `senderid`
Every message sent through MailChannels carries two headers that can be used to identify the `auth` and `senderid` of the message:
- `X-MailChannels-Auth-Id` - this header carries the `auth`
- `X-MailChannels-Sender-Id` - this header carries the `senderid`

### WARNING

Specifying `auth=cloudflare` in your Domain Lockdown record will authorize every Cloudflare Worker to send from your domain. For obvious reasons, this is not recommended.

## References
1) https://blog.cloudflare.com/sending-email-from-workers-with-mailchannels/
2) https://support.mailchannels.com/hc/en-us/articles/200262610-Set-up-SPF-Records
3) https://www.mimecast.com/content/sender-policy-framework/#:~:text=Sender%20Policy%20Framework%20(SPF)%20is,to%20a%20company%20or%20brand.
4) https://support.mailchannels.com/hc/en-us/articles/16918954360845-Secure-your-domain-name-against-spoofing-with-Domain-Lockdown-



## Final notes
This guide is not all-inclusive, and omits some things that I haven't tested or prepared. More information is available on the Cloudflare Blog and MailChannels support site, of which I highly recommend looking through.
This guide exists because of Cloudflare's very lacklustre documentation surrounding their Workers platform. I've been using Workers for nearly two years, and while they have come a long way functionally, I believe that the documentation (or rather lack thereof) has crippled this platform's ability to be widely recommendable. 
Unless some divine inspiration strikes me, I'm unlikely to continue this "improved docs" project outside of just this module. 

If you're a Cloudflare employee viewing this, hi! What you guys are doing over there is truly remarkable and I'm a big fan of it, but please take some time to improve your documentation. I can't say that my documentation is the shining example of what it should be, but I'd love to provide some notes for what I want to see, should you be interested.

If you've found any issues in this guide, or otherwise want to make suggestions or contribute, feel free to do so! I want to make this guide the one-stop-shop for this module, as I believe that is sorely missing from the official documentation.
