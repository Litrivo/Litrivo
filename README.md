# Litrivo – Fuel prices within your range

This is the beginning of a brand new app for petrol stations searching and their comparison
in order to find the cheapest fuel for you. I decided to name this app "Litrivo" as per domain
and trademark research I conducted with ChatGPT and Claude Code, where the name comes out as
a compound of the word for litre, and the -ivo suffix.

This project is being developed as part of my diploma thesis, which continues off my bachelor
thesis actually. While my bachelor thesis focused on proof of concept web app, this project
is far more serious. The idea is to actually serve a fully functional web and native mobile
app for petrol stations and their prices comparison beyond just a diploma project. This app
will be published on Google Play Store as well as on its own domain: litrivo.com; This domain
is already registered, together with litrivo.cz, litrivo.sk, and litrivo.eu, which will all
point to the canonical .com domain.

## Project parts

This project is maintained in several parts unified under a single monorepo, which you are
currently reading.

### Website and web app

This part is the desktop and mobile frontend on a live server, being served as a PWA page.
While Android will have its own dedicated native app, iOS support is not planned at the moment
due to agressive nature of Apple's ecosystem for developers. iPhones will however still
support the PWA version of my app, so iPhone users can simply install it from their browsers
at the app's frontpage.

### Android app

This part is the native app shell for the dedicated Android app support with the use of API
of the live server to connect to the database, with authentication for crowdsourcing.

### API

This is the bridge between frontend and backend servers, and the database. The main role
for this part is to provide functionality for both the web app and the Android app.

### Database

This is the memory of the app. It contains the database of registered users for contributions,
the database of indexed petrol station brands, the database of crowdsourced fuel prices, as well
as the database of specific indexed petrol station locations for each brand.

### Crawler

This is the backend web crawler, not much different from Google, whose ultimate purpose is
to just index and log all of the petrol station brands and their locations for the purpose
of some level of automation in the crowdsourcing process. It doesn't scrape the data of fuel
prices at given locations automatically, it merely just logs the URLs of the pages to find
the petrol stations and saves them in the database alongside with the name of the brand and
the location of the station.

### Server

This is the public web facing server that serves over HTTP and HTTPS ports (80 and 443). It's
ultimately just a reverse proxy for the API and the web app endpoints.

### Kubernetes

This is the Kubernetes cluster with the definition of the workers for this app and all of its
parts minus the dedicated Android native app. This is what is being run over the canonical
domain.

## License

This project is built using the [MIT License](LICENSE). It is released as open-source under
the assumption of good faith, as I believe that software should be accessible and transparent.
It also serves as a sort of documentation for the entire project and is required by the diploma
thesis publication which also will host a copy of the source code at the time of submission.

## AI assistance transparency

I believe in transparency, so I should also state that this project will be AI assisted.
I personally don't believe that the use of AI inherently makes the end result slop. First and
foremost, AI is a tool, and as such can be abused or misused. If it's used by an untrained
individual who's just messing around, the end result will inevitably be slop. When AI is used
by a skilled individual who has a clear vision and actually verifies and tests the output,
the end result can be an actual useful app. Just because a piece of software is tailored with
the use of AI doesn't mean it's slop. If I never told you, or if anyone never told you, and the
app was actually great, would you know it's made with the use of AI? Because I value transparency
and because I believe people should know what they're dealing with, here I am. It's also one of
the requirements of the GitHub and Google Play Store guidelines to disclose the use of AI,
so that's another reason to write this transparency statement. Welcome to the new digital age.
