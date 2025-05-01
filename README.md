# tomfos.tr

This repository contains the source code for my personal homepage, available at
[tomfos.tr](https://tomfos.tr).

The repository itself is hosted on GitHub at
[https://github.com/tcpipuk/tomfos.tr](https://github.com/tcpipuk/tomfos.tr).

## Technology

The site is built using [MkDocs](https://www.mkdocs.org/) with the
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme. The configuration can be
found in `mkdocs.yml`.

## Deployment

The site is automatically built and deployed to [Cloudflare Pages](https://pages.cloudflare.com/)
whenever changes are pushed or merged to the main branch. This process is managed by the GitHub
Actions workflow defined in `.github/workflows/deploy.yml`.

## Development

To run the site locally for development:

1. Ensure you have MkDocs and the Material theme installed. See the
   [MkDocs Material installation guide](https://squidfunk.github.io/mkdocs-material/getting-started/#installation).
2. Clone the repository.
3. Serve the site using the command:

    ```bash
    mkdocs serve
    ```

4. Open your browser to `http://127.0.0.1:8000/`.

## License

The content of this site is licensed under the Apache License, Version 2.0. See the `LICENSE` file
for details.
