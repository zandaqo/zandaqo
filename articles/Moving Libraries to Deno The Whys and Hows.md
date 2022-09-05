# Moving Libraries to Deno: The Whys and Hows

A cursory look at the numerous introductory articles about Deno[^1] will tell you that it’s a Node.js replacement shunning NPM, with built-in TypeScript, and security by the way of permission flags. The underlying message often hints at incompatibility with the existing Node.js ecosystem, emphasizes the “let’s start from scratch” approach, and completely misses the point. Frankly, I would argue that these are not even the main features of Deno and juxtaposing it against the existing Node.js ecosystem is fundamentally misguided.

I have been keeping an eye on Deno since its introduction and last year re-wrote my isomorphic JavaScript libraries in it[^2][^3][^4]. Drawing on that experience, I would like to convince you that Deno is not just a promising platform, but already a capable environment for writing isomorphic JavaScript targeting browsers, Deno, and Node.js alike.

*Disclaimer*: I am not affiliated with Deno the company, and while I am open to bribes, I have not received any from them (yet). Opinions are my own and do not express the views of the rest of humanity.

## A Case for Deno

Sharing code between the browser and the server, the so-called *isomorphic JavaScript*, has been one of the main advantages brought by Node.js. It reduces the amount of code, but more importantly, it reduces the context switch we otherwise have to make when dealing with multiple languages. Unfortunately, due to historical reasons, we now have different APIs performing similar tasks in Node.js and the browsers. The process of reconciliation has been slow and painful, cue ESM modules. Deno, on the other hand, relies on the standard web APIs potentially eliminating the context switch. You can even check in MDN how each web API is supported by Deno.

Adherence to the standard APIs is all the more important since the web is no longer strictly demarcated between clients and servers: we now have Service Workers that are effectively “servers” living on the client machine, and edge computing in workers like Cloudflare Workers sitting in-between the two — all implementing subsets of the standard web APIs.

If you have heard anything about JavaScript in the last decade, you are likely familiar with *the fatigue* — a doomsday scenario where a rogue `node_modules` folder consumes all available space on the net... I jest, of course, yet Jest, the popular test runner, pulls in ~300 dependencies, other tools and frameworks are not far behind. By itself the number is not an issue, however, each such tool exponentially increases the complexity of the toolchain making it harder to set up and maintain. On top of that, every time there is a scandal with some obscure string padding library in NPM, you are guaranteed to take part in the excitement by finding the culprit among your dependencies, often through a developer tool.

Deno addresses this issue in several ways. Firstly, Deno renames the folder in question into `cache` drastically curbing its destructive potential. Secondly, Deno introduces a standard library[^5], think lodash meets Node.js APIs. And finally, Deno has a built-in toolchain...

## The Deno Toolchain

Out of the box, Deno comes with a formatter, a linter, a test runner, a coverage reporter, a bundler, a doc generator, and Ryan knows what else, all a single console command away. This is a huge improvement in developer experience. To produce quality Typescript code, you no longer need to install Prettier, ESLint, Jest, TS Compiler, Webpack, and spend the rest of the day trying to make them all work together through various plugins, presets, and ritualistic dances.

The tools themselves are for the most part written in Rust and usually an order of magnitude faster than their JavaScript counterparts. While some are written from scratch, like `deno_lint`[^6], others are adopted from existing Rust projects, e.g. the formatter uses `dprint`[^7], various compilation and bundling tools rely on `swc`[^8]. That said, Deno still includes `tsc` for type checking simply due to the lack of alternatives.

An underappreciated feature of the Deno toolchain is the opinionated, zero-config, approach it takes. It adopts the de-facto standards of the industry and nudges users into following them. Thus, for example, the configuration options for the formatter and the linter are deliberately limited, although, there is always a way to opt-out if necessary.

In practice, insuring code quality in your project boils down to a handful of one-liners with no external dependencies:

```
# check formatting
deno fmt --check
# run the linter
deno lint
# run tests collecting coverage
deno test -A --coverage=cov
# check coverage
deno coverage cov
```

Admittedly, the tools might not be as feature-rich as their popular counterparts, and at times they are rough around the edges. The linter likely lacks a rule or two from your custom ESLint ruleset. The formatter, while on par with Prettier, for the most part, can sometimes surprise you, for example, right now it has a peculiar way of formatting HTML inside string templates such as the ones used by Lit and HyperHTML.

## Cross-Compiling and Distribution

Code written in Deno can be compiled and bundled for the browser or Node.js consumption with the built-in bundler. However, if you decide to use Typescript code written for Deno as it is with the TypeScript Compiler (`tsc`), e.g. in our usual Node.js toolchains, you will encounter a small but thorny issue with the file imports. Deno (like browsers) imports exactly what you tell it to, it expects full path (extension and all) to the file being imported, be it a URL or a local path:

```
import {...} from "https://deno.land/std@0.123.0/testing/asserts.ts"
import {...} from "./mod.ts"
```

Meanwhile, `tsc` prefers magic and will flag the usage of `.ts` extension in imports. I'll save you the details, but it's one of those situations where `tsc` is wrong and Deno is right. The issue has been raised numerous times with the Typescript team and discussed at length[^9].

There have been workarounds to solve this and other issues that arise when distributing Deno code for Node.js toolchains. Recently, the Deno team introduced `dnt`[^10]-- a build tool that creates an NPM package from Deno code. It is an ambitious tool that solves the imports issue, maps the third-party dependencies you might have to their NPM packages, adds shims for Deno or web-specific APIs you might be using that are not present in Node.js, and much more.

In practice, you can have a single TypeScript (and/or JavaScript) codebase benefiting from the Deno tooling that can then be easily distributed to browsers and NPM. All it takes is a small build script using `dnt` and perhaps a few extra lines in your CI script if you want to automate publishing to NPM as well.

It should also be mentioned that as of this writing the Deno team is busy[^11] improving the compatibility with existing Node.js packages so that even those relying on node-specific APIs can be seamlessly used by Deno. Isomorphic NPM packages have already been widely used through CDNs like esm.sh, Skypack, and the likes.

## Conventions

There are some common practices in Deno that you might find helpful when writing your libraries or simply navigating the ecosystem. These are not enforced by Deno, might at best be considered conventions, and you are free to discard any or all of them:

- The main entry point of a Deno library is usually called `mod.ts`, while `index.ts` is left for applications.
- Third-party dependencies are collected into a file named `deps.ts` or `dev_deps.ts` for dev only dependencies. This way updating versions can be centralized the way it's done in `package.json` for NPM.
- snake_case is used for file names, and test files end with`_test.ts`.
- The folder structure of libraries is flat with most exported files laid out in the main folder and nested folders used for supporting and secondary elements like `docs` for API documentation, `examples` for usage examples, etc. This results in shorter import paths for your dependents since Deno uses URLs for importing.

---

Overall, my experience in porting and maintaining libraries in Deno has been a great improvement over the previous toolchain. Things were not always rosy: debugging in VSCode would occasionally break, coverage was imprecise for a time while testing API was overhauled, importing some node-specific libraries can still be tricky. However, when something does break, it’s much easier to find the cause, or at least whom to blame. Whereas with our usual collection of Node.js tools it often turns into a wild goose chase over a dozen repositories. The Deno team also has been responsive and active in fixing and improving the platform.

---

[^1]: [Deno — A modern runtime for JavaScript and TypeScript](https://deno.land/)
[^2]: [structurae: Data structures for high-performance JavaScript applications.](https://github.com/zandaqo/structurae)
[^3]: [compago: A minimalist framework for modern web applications.](https://github.com/zandaqo/compago)
[^4]: [objectid64: convert MongoDB ObjectIDs into URL-friendly base64.](https://github.com/zandaqo/objectid64)
[^5]: [Deno standard library](https://github.com/denoland/deno_std)
[^6]: [deno_lint: Blazing fast linter for JavaScript and TypeScript written in Rust](https://github.com/denoland/deno_lint)
[^7]: [dprint: Pluggable and configurable code formatting platform written in Rust.](https://github.com/dprint/dprint)
[^8]: [swc: Rust-based platform for the Web](https://github.com/swc-project/swc)
[^9]: [allow voluntary .ts suffix for import paths · Issue #37582 · microsoft/TypeScript](https://github.com/microsoft/TypeScript/issues/37582)
[^10]: [dnt: Deno to npm package build tool.](https://github.com/denoland/dnt)
[^11]: [Improve Node Compat Mode · Issue #12577 · denoland/deno](https://github.com/denoland/deno/issues/12577)