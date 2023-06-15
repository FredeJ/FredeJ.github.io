Title: Starting typescript, webpack and other impossible tasks
Date: 15-06-2023
Category: Rant
Status: Draft

I'll put it right up front: This is a rant. This is what I'm doing instead of working on what I wanted to work on.

I'm writing this rant because I'm once again stopped in my tracks on a project because of typescript and webpack. Every time I go in with the best of intentions of doing things _right_ this time and getting some stuff done. And every time I end up debugging webpack or having to spend hours understanding some strange configuration option, which ends up taking all of my alotted time. And by the end of it, I still haven't fixed the issue.

First of, I much prefer strongly typed languages.  My favourite language is C++. If I wasn't using C++ I would likely use Rust - But I genuinely like working in C++! Often, I don't even have the choice and sometimes I even have to work in C.

So, obviously I would choose Typescript, whenever I had the choice.

The world is simple in C and C++ - not easy, but simple. You have a compiler. It compiles your files to binary. You link the binary files together, to make a program. There's a bit more to it, but if you understand operating systems and hex addresses for registers don't scare you, it's pretty straight forward. Not that you need to use that stuff - it's just that it's kind of the same understanding. Obviously there are build systems for when you need to go bigger, but CMake is generally pretty okay. It's basically: There are these files, they fit together like this and when you're done call it this.

Enter Javascript and Typescript.

With Typescript, you write your .ts files and you compile them with `tsc`. So far so good. Your output is a .js file. Okay, I get it, javascript is used everywhere. It's sort of fine as an intermediate medium, but it's confusing that it's a human readable file that's you can go and change. You shouldn't, obviously. But that brings me to my first issue: What do I link to in my html file? Is it `<script src="src/index.ts">` ? Or should it be `<script src="dist/index.js">`? No, it's probably just `<script src="index.js">` no I mean `<script src="index.ts">` because i'm writing my raw files here, that are compiled, right?

Okay, so that's an additional layer of complexity I need to deal with when I choose Typescript.

But okay, I'm willing to deal with it if I get my strong types and all the other stuff Typescript promises. And as long as my compiler handles that stuff for me, it's fine right? Wait, are html files compiled? Not with `tsc` I think. Okay, so I need something like webpack to handle that for me. I'll set that up.

Okay, great, I'm pumped, let's get started, shall we?

I create my project, I choose Typescript when asked. Awesome, this is going great. Finished setting up, let's run it so we can get started aaaaand it fails. Cant run. Says I didn't properly configure \_proxyProfileProviderRenderer or whatever. This has happened multiple times to me - that whatever default project fails to run with Typescript.

Anyways, I press on. Okay, that's a known issue. It's a pretty simple fix. Just add `"renderer": "local"` or whatever. Which brings me to my next point.

Everything has its own god damn configuration file, using their own god damn method of configuring. You need a `package.json` which is a json object, a `tsconfig.json` which is obviously also a json object, a `.eslint` file which, turns out, is also a json object, a `webpack.config.js` which is a script, exporting a json object. 

Now add `"renderer": "local"`. Somewhere.

Anyways.

Let's say I'm building an electron app. We say that, because that's exactly what I wanted to be building right now instead of writing this stupid blog post. All the other stuff were really past frustrations. This is current frustration. Current as in, I have two windows open right now. A window with a blog post and a window with frustration. And I'm choosing to focus on the blog post because I can't deal with the frustration anymore.

To start of, we grab the `electron-quick-start-typescript` repository. It runs right out of the box - great! I can edit things and compile and see things in action. Buuut it does look like it will be a pretty slow process without hot reloading. Hot reloading in electron - that must be a solved problem. I'll just google that real quick. Yep, just as I figured, `npm install electron-reload`, just load the library and add a function to the bottom of the page and we're golden. Nope. Okay, didn't work. Eh, it's probably just an abandonded project, what else is there? `npm install electron-reloader`. Yeah, this one must be it. I'll add it to the page and .. Hmm, what's `fs-events`? And why is it missing? Okay, it's a Linux thing that's loaded by mistake on Windows. I can just disable it by adding `DontLoadPlugin` and creating a `plugins` list in `webpack.config.js`. Great. Doesn't complain anymore. Doesn't work either though. Ahhh. Fuck this. Here's one. An article about "How to do hot reloading in electron with no dependencies". Perfect. That's exactly what I need. I hate dependencies. Give it to me raw.

Man, it's pretty simple. Just create `watcher.js` and add a pretty basic function. I mean, `watcher.ts`, you know, we use types around here. Oh, doesn't work in Typescript. But that's fine, I can probably modify that a bit, shouldn't be too hard. There. That should do it. And compile and nothing. Wait, shouldn't it turn it into a .js file?

Let's check out the config. 

```javascript
// webpack.config.js
module.exports = [
{
        mode: 'development',
        entry: './src/renderer.ts',
        target: 'electron-renderer',
        plugins: [
            ...optionalPlugins,
        ],
        module: {
            rules: [
                {
                    test: /\.ts$/,
                    include: [/src/],
                    use: [{ loader: 'ts-loader' }]
                }
            ]
        },
        output: {
            path: __dirname + '/dist',
            filename: 'renderer.js'
        },
        resolve: {
            extensions: ['.ts', '.js'],
            modules: ['node_modules', path.resolve(__dirname + 'src')]
        }
    },
    ...
]
```

To me, this looks like it should find anything called `.ts` or `.js` in the `src/` folder and sort of kind of include it in the final file. But no no, that's not how any of this works. You know, it probably needs to find the `watcher.ts` file directly in the `renderer.ts` file. Lets just include it, oh no, it's not a module, sorry.

Come to think of it, it does not look like it's set up to handle HTML files in any way. Maybe that's the issue. I don't know anymore. How do I configure that? I'll have to dig into the documentation. What was the project I was working on, again?

## What to do?

Okay, fair enough, each of the components i've mentioned here basically do their job pretty well on their own. Like, I get that the `tsc` people just want to design a nice language and build a nice compiler. And they do and it works and it's great. I get that `eslint` wants to be able to run linting anywhere, and just read a simple file to figure out how to do it. I get that webpack is so flexible that it can do absolutely everything for everyone. And that it's a pluggable system where you set it up and use only what you need. And that none of these wants to depend on each other so of course they get their own config file.

I get that the people who are developing the small loader modules don't owe me anything and I'm just happy that they're willing to share their work. Even if it doesn't work or it's outdated or whatever other reason it's like that.

And I get that part of the strength of the JS ecosystem is the ability for people to build their own frameworks and tools (and oh boy are they necessary).

But I'm left here with a shitty developer experience. Again.

And I can't help but feel that with this abundance of tools that all solve the same things in slightly different ways, would be helped tremendously by having a basic set of tools, guaranteed to do some things by a standard. Have a reference packer that does some things but not all. Enable hot reloading by default or have a super simple way of turning it on. Provide a basic linter and a default, opinionated formatting configuration.

Everywhere I've seen that standards are provided and enforced, it has made for a better developer experience. Go and Rust come to mind. Javascript / Typescript should follow.