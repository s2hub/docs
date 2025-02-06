# How to redirect a website to another domain except for the backend

Sometimes projects need to be moved to a new domain. This can be a simple redirect from the old domain to the new one. But what if you want to keep the backend on the old domain? Maybe you still want to have access to all the files and images. Or if you have a headless Silverstripe CMS and the frontend is on a different domain... there are many use cases.

The follwing `.htaccess` snippet will do the job:

```apache
RewriteCond %{REQUEST_URI} !^/(admin|assets|resources|Security)/ [NC]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule ^(.*)$ https://otherdomain.com/$1 [R=301,L]
```

This snippet will redirect all requests to the new domain, except for existing files and the admin, assets, resources and Security folders. This way, you can still access the backend on the old domain.