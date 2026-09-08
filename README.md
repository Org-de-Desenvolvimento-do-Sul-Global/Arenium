<div align="center">

<img height="200" src="./assets/Arenium-256x256.png" alt="Arenium logo">

# Arenium

[![Nostr](https://img.shields.io/badge/nostr-purple.svg?style=for-the-badge\&logo=data:image/svg%2bxml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNTYgMjU2Ij48cGF0aCBmaWxsPSIjZmZmIiBkPSJNMjEwLjggMTk5LjRjMCAzLjEtMi41IDUuNy01LjcgNS43aC02OGMtMy4xIDAtNS43LTIuNS01LjctNS43di0xNS41Yy4zLTE5IDIuMy0zNy4yIDYuNS00NS41IDIuNS01IDYuNy03LjcgMTEuNS05LjEgOS4xLTIuNyAyNC45LS45IDMxLjctMS4yIDAgMCAyMC40LjggMjAuNC0xMC43cy05LjEtOC42LTkuMS04LjZjLTEwIC4zLTE3LjctLjQtMjIuNi0yLjQtOC4zLTMuMy04LjYtOS4yLTguNi0xMS4yLS40LTIzLjEtMzQuNS0yNS45LTY0LjUtMjAuMS0zMi44IDYuMi40IDUzLjMuNCAxMTYuMXY4LjRjMCAzLjEtMi42IDUuNi01LjcgNS42SDU3LjdjLTMuMSAwLTUuNy0yLjUtNS43LTUuN3YtMTQ0YzAtMy4xIDIuNS01LjcgNS43LTUuN2gzMS43YzMuMSAwIDUuNyAyLjUgNS43IDUuNyAwIDQuNyA1LjIgNy4yIDkgNC41IDExLjQtOC4yIDI2LTEyLjUgNDIuNC0xMi41IDM2LjYgMCA2NC40IDIxLjQgNjQuNCA2OC43djgzLjJaTTE1MCA5OS4zYzAtNi43LTUuNC0xMi4xLTEyLjEtMTIuMXMtMTIuMSA1LjQtMTIuMSAxMi4xIDUuNCAxMi4xIDEyLjEgMTIuMVMxNTAgMTA2IDE1MCA5OS4zWiIvPjwvc3ZnPg==)](https://github.com/nostr-protocol/nostr)
[![Svelte](https://img.shields.io/badge/Svelte-FF3E00?style=for-the-badge\&logo=svelte\&logoColor=white)](https://svelte.dev/)
[![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-06B6D4?style=for-the-badge\&logo=tailwindcss\&logoColor=white)](https://tailwindcss.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge\&logo=postgresql\&logoColor=white)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge\&logo=docker\&logoColor=white)](https://www.docker.com/)

<i>Governance as a Service</i>

</div>

## Description

A decentralized governance system based on [Nostr Protocol](https://github.com/nostr-protocol/nostr). Each user has their own cryptographic identity (a public/private key pair), and data travels across a network of independent relays. Host your own relay or connect to public ones.

## Supported NIPs

| NIP | Description |
|-----|-----------|
| NIP-01 | Basic protocol |
| NIP-04 | Encrypted direct messages |
| NIP-05 | Identity verification via DNS |
| NIP-07 | Signing via browser extension |
| NIP-19 | Bech32 identifiers (npub, nsec, note) |
| NIP-29 | Relay-based Groups |
| NIP-88 | Polls |

## Instructions


### Hosting a Dedicated Relay

Make sure to have Docker installed

### Installation

Clone the repository:

```bash
git clone https://github.com/Org-de-Desenvolvimento-do-Sul-Global/Arenium.git
cd Arenium
```

Install the dependencies:

```bash
npm install
```

Configure the necessary environment variables in the `.env` file:

```env
DATABASE_URL=<postgresql-connection-string>
```

Configure the PostgreSQL database as needed.

### Execution

Start the development server:

```bash
npm run dev
```

### Build

To generate a production build:

```bash
npm run build
```

To run the built version:

```bash
npm run preview
```

## How to contribute

Contributions are welcome! To contribute:

1. Fork the project
2. Create a feature branch (git checkout -b feature/new-feature)
3. Commit your changes (git commit -m 'Add new feature')
4. Push to the remote repository (git push origin feature/new-feature)
5. Open a Pull Request

## Reporting bugs

Found a bug? Open an issue describing:

- The expected behavior
- The actual behavior
- Steps to reproduce

## License

This project is licenced under the terms of _ License.

Head over to [`LICENSE`](LICENSE) to obtain the full text of the licence.
