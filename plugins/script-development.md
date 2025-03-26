# Script Development

This guide will help you set up a local development environment for creating and testing in-context scripts. By hosting your script locally, you can quickly iterate and debug your code before deploying it to the in-context platform.

## Setting up a Local Server

You can use a simple HTTP server to serve your script files. For most cases, this is more than enough. If your script development requires more complex setups with multiple files and bundling, Vite is a good alternative. Here are two popular options:

### 1. http-server

`http-server` is a simple, zero-configuration command-line HTTP server.

**Installation:**

```bash
npm install -g http-server
```

**Usage:**

1.  Navigate to the directory containing your script file.
2.  Run `http-server -c-1 --cors`. This will start a server, usually on port 8080.

    - `-c-1` disables caching, which is helpful during development.
    - `--cors` enables Cross-Origin Resource Sharing (CORS), which is required for loading the script from `localhost`.

Your script will then be accessible at `http://localhost:8080/your-script.js`.

### 2. Vite

Vite is a fast, modern build tool that also includes a development server with hot module replacement.

**Installation:**

```bash
npm install -g create-vite
npm create-vite my-incontext-script --template vanilla
cd my-incontext-script
npm install
```

**Usage:**

1.  Replace the contents of `my-incontext-script/src/main.js` with your script code.
2.  Run `npm run dev`. This will start a development server, usually on port 3000.

Your script will then be accessible at `http://localhost:3000/src/main.js`. Note that Vite often serves files from a `src` directory.

## Configuring your Script in in-context

In your in-context plugin configuration, set the `src` property of your script to point to your local server:

```json
{
	"scripts": [
		{
			"src": "http://localhost:8080/your-script.js" // or http://localhost:3000/src/main.js for Vite
		}
	]
}
```
