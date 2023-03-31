# hacks
hacks
"dotenv-flow": "^3.2.0",
		"express": "^4.18.2",
		"express-http-proxy": "^1.6.3",
		"rammerhead": "https://github.com/holy-unblocker/rammerhead/releases/download/v1.2.41-holy.1/rammerhead-1.2.41-holy.1.tgz",
		"rammerhead": "https://github.com/holy-unblocker/rammerhead/releases/download/v1.2.41-holy.3/rammerhead-1.2.41-holy.3.tgz",
		"rimraf": "^3.0.2",
		"website": "https://github.com/holy-unblocker/website-embedded/releases/download/v1.0.0-holy.2/website-1.0.0-holy.2.tgz"
	},
  93  
scripts/start.js
Comment on this file
@@ -5,24 +5,18 @@ import { expand } from 'dotenv-expand';
import { config } from 'dotenv-flow';
import express from 'express';
import proxy from 'express-http-proxy';
import { createRequire } from 'module';
import { fork } from 'node:child_process';
import { createServer } from 'node:http';
import { join } from 'node:path';
import createRammerhead from 'rammerhead/src/server/index.js';
import { websitePath } from 'website';

const require = createRequire(import.meta.url);

// what a dotenv in a project like this serves: .env.local file containing developer port
expand(config());

console.log(`${chalk.cyan('Starting the server...')}\n`);

const app = express();
const rh = createRammerhead();

// used when forwarding the script
const rammerheadScopes = [
	'/([a-z0-9]{32})*',
	'/rammerhead.js',
	'/hammerhead.js',
	'/transport-worker.js',
@@ -39,6 +33,12 @@ const rammerheadScopes = [
	'/api/shuffleDict',
];

const rammerheadSession = /^\/[a-z0-9]{32}/;

console.log(`${chalk.cyan('Starting the server...')}\n`);

const app = express();

app.use(
	'/api/db',
	proxy(`https://holyubofficial.net/`, {
@@ -69,6 +69,8 @@ const bare = createBareServer('/api/bare/');
server.on('request', (req, res) => {
	if (bare.shouldRoute(req)) {
		bare.routeRequest(req, res);
	} else if (shouldRouteRh(req)) {
		routeRhRequest(req, res);
	} else {
		app(req, res);
	}
@@ -77,6 +79,8 @@ server.on('request', (req, res) => {
server.on('upgrade', (req, socket, head) => {
	if (bare.shouldRoute(req)) {
		bare.routeUpgrade(req, socket, head);
	} else if (shouldRouteRh(req)) {
		routeRhUpgrade(req, socket, head);
	} else {
		socket.end();
	}
@@ -163,64 +167,43 @@ while (true) {

		console.log('');

		setTimeout(() => startRammerhead(), 30e3);

		break;
	} catch (err) {
		console.error(chalk.yellow(chalk.bold(`Couldn't bind to port ${port}.`)));
	}
}

async function startRammerhead() {
	// start rammerhead late so the first port is always Holy Unblocker
	const rhPort = await findPort();

	fork(require.resolve('rammerhead/bin.js'), {
		stdio: ['ignore', 'ignore', 'inherit', 'ipc'],
		env: {
			...process.env,
			PORT: rhPort,
		},
	});

	const rammerheadProxy = proxy(`http://127.0.0.1:${rhPort}`, {
		proxyReqPathResolver: (req) =>
			req.originalUrl.replace(/^\/[a-z0-9]{32}\/.*?:\/(?!\/)/, '$&/'),
	});

	for (const url of rammerheadScopes) app.use(url, rammerheadProxy);
}

function randomPort() {
	return ~~(Math.random() * (65536 - 1024 - 1)) + 1024;
}

function tryBind(port) {
	return new Promise((resolve, reject) => {
		// http server.. net server, same thing!
		const server = createServer();

		server.on('error', (error) => {
			reject(error);
		});

		server.on('listening', () => {
			server.close(() => resolve());
		});

		server.listen({ port });
	});
/**
 *
 * @param {import('node:http').IncomingRequest} req
 */
function shouldRouteRh(req) {
	const url = new URL(req.url, 'http://0.0.0.0');
	return (
		rammerheadScopes.includes(url.pathname) ||
		rammerheadSession.test(url.pathname)
	);
}

async function findPort() {
	while (true) {
		const port = randomPort();
/**
 *
 * @param {import('node:http').IncomingRequest} req
 * @param {import('node:http').ServerResponse} res
 */
function routeRhRequest(req, res) {
	rh.emit('request', req, res);
}

		try {
			await tryBind(port);
			return port;
		} catch (err) {
			// try again
		}
	}
/**
 *
 * @param {import('node:http').IncomingRequest} req
 * @param {import('node:stream').Duplex} socket
 * @param {Buffer} head
 */
function routeRhUpgrade(req, socket, head) {
	rh.emit('upgrade', req, socket, head);
}
