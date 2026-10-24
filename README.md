# Your Portfolio Site

A static site for your architecture / urban / computational design work.
Host it free with GitHub Pages.

## Setup (one-time, ~10 minutes)

1. Create a free GitHub account at https://github.com
2. Create a new repository named **YOUR-USERNAME.github.io**
   (must exactly match your username)
3. Upload all files in this folder to that repo
   (on the repo page: *Add file → Upload files* — drag everything in)
4. Go to *Settings → Pages* → Source: *Deploy from a branch* → Branch: *main*, folder */ (root)* → Save
5. Wait 1–2 minutes. Your site is live at **https://YOUR-USERNAME.github.io**

## Editing content

- `index.html` — homepage, project cards. Copy a card block to add a project.
- `projects/*.html` — one page per project.
- `styles.css` — all styling.
- Replace `YOUR NAME` and `YOUR-USERNAME` with your actual details.

## Adding images

1. Put images in a folder, e.g. `img/`
2. Reference them: `<img src="../img/my-plan.png" alt="Ground floor plan">`
3. Or replace a `<div class="placeholder">…</div>` with a real `<figure>` block

## Adding a new project

1. Duplicate a file in `projects/`, rename it, edit the text
2. Add a matching card in `index.html`


## Attribution (keep it honest)

If a project followed a tutorial or was scripted with AI help, say so
on the project page and link the source. It reads as *process literacy*,
not weakness — reviewers care that you know what you built and how.
