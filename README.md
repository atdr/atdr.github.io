# atdr.github.io

Forwards `atdr.github.io` to [atdr.uk](https://atdr.uk), whose source is [atdr/atdr.uk](https://github.com/atdr/atdr.uk). `index.html` and `404.html` send each path to the same path on atdr.uk, since GitHub Pages can't send HTTP redirects.

Project sites such as [atdr.github.io/capsule-caffeine](https://atdr.github.io/capsule-caffeine/) are served by their own repositories and are not affected. This repository must not set a custom domain: a user site's custom domain applies to every project site without its own, which would move them all under atdr.uk.
