# Paperbark Farm — staff dashboard

Single-file static dashboard served via GitHub Pages. All data lives in Supabase
behind Supabase Auth + row-level security (`is_staff()`); this page is just the
login screen and UI. The embedded key is the publishable (public) API key.

Source of truth for this file: `dashboard/index.html` in the private
`paperbark-podio-migration` repo — edit there, then copy here and push to deploy.
