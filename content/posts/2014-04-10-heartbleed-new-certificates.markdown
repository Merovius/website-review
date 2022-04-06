---
date: "2014-04-10T21:28:25Z"
tags:
- meta
aliases:
- "/2014/04/10/heartbleed-new-certificates.html"
omitFeed: true
title: 'Heartbleed: New certificates'
summary: Updating my TLS-certificates due to Heartbleed.
---
Due to the Heartbleed vulnerability I had to recreate all TLS-keys of my
server. Since CACert appears to be mostly dead (or dying at least), I am
currently on the lookout for a new CA. In the meantime I switched to
self-signed certificates for all my services.

The new fingerprints are:
<table>
	<tr>
		<th>Service</th>
		<th>SHA1-Fingerprint</th>
	</tr>
	<tr>
		<td>merovius.de</td>
		<td>8C:85:B1:9E:37:92:FE:C9:71:F6:0E:C6:9B:25:9C:CD:30:2B:D5:35</td>
	</tr>
	<tr>
		<td>blog.merovius.de</td>
		<td>1B:DB:45:11:F3:EE:66:8D:3B:DF:63:B9:7C:D9:FC:26:A4:D1:E1:B8</td>
	</tr>
	<tr>
		<td>git.merovius.de</td>
		<td>65:51:16:25:1A:9E:50:B2:F7:D7:8A:2B:77:DE:DE:0C:02:3C:6C:ED</td>
	</tr>
	<tr>
		<td>smtp (mail.merovius.de)</td>
		<td>1F:E5:3F:9D:EE:B4:47:AE:2E:02:D8:2C:1E:2A:6C:FC:D6:62:99:F4</td>
	</tr>
	<tr>
		<td>jabber (merovius.de)</th>
		<td>15:64:29:49:82:0E:8B:76:47:1A:19:5B:98:6F:E4:56:24:D9:69:07</td>
	</tr>
</table>

This is of course useless in the general case, but if you already trust my
gpg-key, you can use

```sh
curl http://blog.merovius.de/2014/04/10/heartbleed-new-certificates.html | gpg
```

to get this post signed and verified.
