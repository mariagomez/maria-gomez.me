---
layout: none
permalink: .htaccess
---
RewriteEngine On
RewriteCond %{HTTP_REFERER} blackhatworth.com [NC,OR]
RewriteCond %{HTTP_REFERER} bestwebsitesawards.com [NC,OR]
RewriteCond %{HTTP_REFERER} hulfingtonpost.com [NC,OR]
RewriteCond %{HTTP_REFERER} buttons-for-website.com [NC,OR]
RewriteCond %{HTTP_REFERER} semalt.semalt.com [NC,OR]
RewriteCond %{HTTP_REFERER} darodar.com [NC,OR]
RewriteCond %{HTTP_REFERER} priceg.com [NC,OR]
RewriteCond %{HTTP_REFERER} ilovevitaly.co [NC,OR]
RewriteCond %{HTTP_REFERER} make-money-online.7makemoneyonline.com [NC]
RewriteRule .* – [F]
