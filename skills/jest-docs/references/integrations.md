# Integrations and platform packages (Jest 30.5)

## Contents

- React Native
- Puppeteer
- MongoDB and DynamoDB
- Platform packages

## React Native

- Projects created with `npx @react-native-community/cli init` set `preset: 'react-native'`. When it fails to resolve, install `@react-native/jest-preset` and use it. https://jestjs.io/docs/tutorial-react-native#setup
- `react-test-renderer` is deprecated: use `@testing-library/react-native` with its `test-renderer` peer, and await `render`. https://jestjs.io/docs/tutorial-react-native#snapshot-test

```tsx
import {render, screen} from '@testing-library/react-native';
test('renders correctly', async () => {
  await render(<Intro />);
  expect(screen.toJSON()).toMatchSnapshot();
});
```

https://jestjs.io/docs/tutorial-react-native#snapshot-test

- The preset is a node environment (no DOM). https://jestjs.io/docs/tutorial-react-native#environment
- Many react-native packages ship untranspiled code. List them in a single `transformIgnorePatterns` pattern: split negative lookaheads cancel each other. https://jestjs.io/docs/tutorial-react-native#transformignorepatterns-customization

```json
{"transformIgnorePatterns": ["node_modules/(?!((@)?react-native|my-project)/)"]}
```

https://jestjs.io/docs/tutorial-react-native#transformignorepatterns-customization

- Stub native components with `jest.mock('react-native-video', () => 'Video')`, and put repeated mocks in a `setupFiles` file. https://jestjs.io/docs/tutorial-react-native#mock-native-modules-using-jestmock

## Puppeteer

- The `jest-puppeteer` preset (`"preset": "jest-puppeteer"`) exposes `page` and `browser`. https://jestjs.io/docs/puppeteer#use-jest-puppeteer-preset
- Code passed to `page.evaluate`, `page.$eval` or `page.$$eval` gets no coverage. https://jestjs.io/docs/puppeteer
- Manual wiring: launch the browser in `globalSetup` and write its `wsEndpoint` to a file (globals from setup do not reach tests). Connect in the environment `setup()`, disconnect in `teardown()`, close in `globalTeardown`. https://jestjs.io/docs/puppeteer#custom-example-without-jest-puppeteer-preset

```js
class PuppeteerEnvironment extends require('jest-environment-node').TestEnvironment {
  async setup() {
    await super.setup();
    const wsEndpoint = await readFile(path.join(DIR, 'wsEndpoint'), 'utf8');
    this.global.__BROWSER_GLOBAL__ = await puppeteer.connect({browserWSEndpoint: wsEndpoint});
  }
  async teardown() {
    if (this.global.__BROWSER_GLOBAL__) this.global.__BROWSER_GLOBAL__.disconnect();
    await super.teardown();
  }
}
```

https://jestjs.io/docs/puppeteer#custom-example-without-jest-puppeteer-preset

## MongoDB and DynamoDB

- MongoDB: preset `@shelf/jest-mongodb` provides `globalThis.__MONGO_URI__` and `globalThis.__MONGO_DB_NAME__`. Close the connection in `afterAll`. https://jestjs.io/docs/mongodb#use-jest-mongodb-preset
- DynamoDB: preset `@shelf/jest-dynamodb` with a `jest-dynamodb-config.js` exporting `{tables: [...]}`. Point the client at the local endpoint when `JEST_WORKER_ID` is set. https://jestjs.io/docs/dynamodb#use-jest-dynamodb-preset

```js
const isTest = process.env.JEST_WORKER_ID;
const config = {convertEmptyValues: true, ...(isTest && {endpoint: 'localhost:8000', sslEnabled: false, region: 'local-env'})};
```

https://jestjs.io/docs/dynamodb#use-jest-dynamodb-preset

## Platform packages

- `jest-changed-files`: `getChangedFilesForRoots(['./'], {lastCommit: true})`. https://jestjs.io/docs/jest-platform#jest-changed-files
- `jest-diff`: `const {diff} = require('jest-diff')`. https://jestjs.io/docs/jest-platform#jest-diff
- `jest-docblock`: `parseWithComments` returns `comments` and `pragmas`. https://jestjs.io/docs/jest-platform#jest-docblock
- `@jest/get-type`: `const {getType} = require('@jest/get-type')`. https://jestjs.io/docs/jest-platform#jest-get-type
- `jest-validate`: `validate(config, {comment, exampleConfig})` returns `{hasDeprecationWarnings, isValid}`. https://jestjs.io/docs/jest-platform#jest-validate
- `jest-worker` runs module exports as promise-returning methods in forked processes. https://jestjs.io/docs/jest-platform#jest-worker
- `pretty-format`: `const {format: prettyFormat} = require('pretty-format')`. https://jestjs.io/docs/jest-platform#pretty-format
- Community packages: the jest-community organization lists `vscode-jest`, `jest-extended`, `eslint-plugin-jest` and `awesome-jest`. https://jestjs.io/docs/jest-community
