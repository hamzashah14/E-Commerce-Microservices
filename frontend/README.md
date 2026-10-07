# Frontend

React 19 + TypeScript storefront built with Create React App (`react-scripts` 5) and Material UI. Node.js 20+ is required.

## Scripts

| Command | Effect |
|---------|--------|
| `npm start` | development server on http://localhost:3000 |
| `npm run build` | production build into `build/` |
| `npm test` | test runner |

## How the build is used

- **Docker Compose** serves `build/` with `nginx:alpine` (config in `nginx.conf`). Run `npm run build` before `docker compose up`.
- **CI and Kubernetes** use `Dockerfile`: it runs `npm run build` and serves the result with `serve` on port 3000.

## API URL

The API base URL is read from `REACT_APP_API_URL` at **build time** and defaults to `http://localhost:3001/api`. Create React App inlines the value into the bundle, so setting it on a running container has no effect. Rebuild the image to change it. In the cluster the browser reaches the gateway through a port-forward on local port 3001; see the [deployment guide](../docs/deployment-guide.md#access-the-applications).

`src/setupProxy.js` forwards `/api` to `http://localhost:3003` in development when the API URL is relative.
