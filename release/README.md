# Release

### Deploying new MonriJS release to WebPay

#### Pushing new MonriJS release to Webpay::development

1. On MonriJS project checkout to development branch
2. On WebPay project checkout to development branch
3. Run `npm run build-prod` in terminal
4. On WebPay project checkout to new branch named xy/monri-js-release-YYYY-MM-DD
5. Commit with message `MonriJS release YYYY-MM-DD`
6. Push
7. Create PR on WebPay

<pre class="language-json"><code class="lang-json">  "scripts": {
    "test": "npm run unit-test",
    "unit-test": "mocha -r ts-node/register test/**/*.spec.ts",
    "integration-test": "mocha test/integration/*.spec.js",
    "watch:components": "webpack --watch --config components/webpack.config.js",
    "watch:components:local": "webpack --watch --config components/webpack.config.js",
    "build:components": "webpack --config components/webpack.config.js",
    "dev:build:components": "webpack --config components/webpack.config.js",
    "dev:build:form": "webpack --config form/webpack.config.js",
    "dev:build:lightbox": "webpack --config lightbox/webpack.config.js",
    "dev:watch:components": "webpack --watch --config components/webpack.config.js",
    "dev:watch:form": "webpack --watch --config form/webpack.config.js",
    "dev:watch:lightbox": "webpack --watch --config lightbox/webpack.config.js",
    "watch:components-old": "webpack --watch --config components/webpack-old.config.js",
    "watch:components-test": "webpack --watch --config components/test.config.js",
    "watch:components-test-old": "webpack --watch --config components/test-old.config.js",
    "watch:lightbox": "webpack --watch --config lightbox/webpack.config.js",
    "prod:watch:lightbox": "webpack --watch --config lightbox/prod.config.js",
    "watch:form-prod": "webpack --watch --config form/prod.config.js",
    "build:form:dev": "webpack --config form/webpack.config.js",
    "build:form:prod": "webpack --config form/prod.config.js",
    "prod:build:form": "webpack --config form/prod.config.js",
    "prod:build:lightbox": "webpack --config lightbox/prod.config.js",
    "prod:build:components": "webpack --config components/prod.config.js",
    "build:lightbox:prod": "webpack --config lightbox/prod.config.js",
    "build:components:prod": "webpack --config components/prod.config.js",
    "build-dev": " npm-run-all --parallel=true dev:build:*",
    <a data-footnote-ref href="#user-content-fn-1">"build-prod": " npm-run-all --parallel=true prod:build:*",</a>
    "build-test": " npm-run-all --parallel=true test:build:*",
    "watch-test": " npm-run-all --parallel=true test:watch:*",
    "build:form:prod:all": "npm run build:form:prod &#x26; npm run build:form:prod",
    "build:form:dev:all": "npm run build:form:dev &#x26; npm run build:form:extensions:olx",
    "test:build:form": "webpack --config form/test.release.config.js",
    "test:build:components": "webpack --config components/test.release.config.js",
    "test:build:lightbox": "webpack --config lightbox/test.release.config.js",
    "test:watch:form": "webpack --watch --config form/test.release.config.js",
    "test:watch:components": "webpack --watch --config components/test.release.config.js",
    "test:watch:lightbox": "webpack --watch --config lightbox/test.release.config.js"
  },
</code></pre>

<figure><img src="../.gitbook/assets/Screenshot 2024-07-31 at 15.53.22.png" alt=""><figcaption><p>Example of a successfully created build ready for use on WebPay</p></figcaption></figure>

[^1]: <mark style="color:purple;">**Production deployment! :)**</mark>
