<div align="center">

# Node Llama.cpp
Node.js bindings for llama.cpp.

<sub>Pre-built bindings are provided with a fallback to building from source with `node-gyp`.<sub>

[![Build](https://github.com/withcatai/node-llama-cpp/actions/workflows/build.yml/badge.svg)](https://github.com/withcatai/node-llama-cpp/actions/workflows/build.yml)
[![License](https://badgen.net/badge/color/MIT/green?label=license)](https://www.npmjs.com/package/node-llama-cpp)
[![License](https://badgen.net/badge/color/TypeScript/blue?label=types)](https://www.npmjs.com/package/node-llama-cpp)
[![Version](https://badgen.net/npm/v/node-llama-cpp)](https://www.npmjs.com/package/node-llama-cpp)

</div>

## Installation
```bash
npm install --save node-llama-cpp
```

This package comes with pre-built binaries for macOS, Linux and Windows.

If binaries are not available for your platform, it'll fallback to download the latest version of `llama.cpp` and build it from source with `node-gyp`.
To disable this behavior set the environment variable `NODE_LLAMA_CPP_SKIP_DOWNLOAD` to `true`.

## Documentation
### [API reference](https://withcatai.github.io/node-llama-cpp/modules.html)

### Usage
#### As a chatbot
```typescript
import {fileURLToPath} from "url";
import path from "path";
import {LlamaModel, LlamaContext, LlamaChatSession} from "node-llama-cpp";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

const model = new LlamaModel({
    modelPath: path.join(__dirname, "models", "codellama-13b.Q3_K_M.gguf")
});
const context = new LlamaContext({model});
const session = new LlamaChatSession({context});


const q1 = "Hi there, how are you?";
console.log("User: " + q1);

const a1 = await session.prompt(q1);
console.log("AI: " + a1);


const q2 = "Summerize what you said";
console.log("User: " + q2);

const a2 = await session.prompt(q2);
console.log("AI: " + a2);
```

##### Custom prompt handling against the model
```typescript
import {fileURLToPath} from "url";
import path from "path";
import {LlamaModel, LlamaContext, LlamaChatSession, ChatPromptWrapper} from "node-llama-cpp";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

export class MyCustomChatPromptWrapper extends ChatPromptWrapper {
    public override wrapPrompt(prompt: string, {systemPrompt, promptIndex}: {systemPrompt: string, promptIndex: number}) {
        if (promptIndex === 0) {
            return "SYSTEM: " + systemPrompt + "\nUSER: " + prompt + "\nASSISTANT:";
        } else {
            return "USER: " + prompt + "\nASSISTANT:";
        }
    }

    public override getStopStrings(): string[] {
        return ["USER:"];
    }
}

const model = new LlamaModel({
    modelPath: path.join(__dirname, "models", "codellama-13b.Q3_K_M.gguf"),
    promptWrapper: new MyCustomChatPromptWrapper() // by default, LlamaChatPromptWrapper is used
})
const context = new LlamaContext({model});
const session = new LlamaChatSession({context});


const q1 = "Hi there, how are you?";
console.log("User: " + q1);

const a1 = await session.prompt(q1);
console.log("AI: " + a1);


const q2 = "Summerize what you said";
console.log("User: " + q2);

const a2 = await session.prompt(q2);
console.log("AI: " + a2);
```

#### Raw
```typescript
import {fileURLToPath} from "url";
import path from "path";
import {LlamaModel, LlamaContext, LlamaChatSession} from "node-llama-cpp";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

const model = new LlamaModel({
    modelPath: path.join(__dirname, "models", "codellama-13b.Q3_K_M.gguf")
});

const context = new LlamaContext({model});

const q1 = "Hi there, how are you?";
console.log("AI: " + q1);

const tokens = context.encode(q1);
const res: number[] = [];
for await (const modelToken of context.evaluate(tokens)) {
    res.push(modelToken);
    
    // it's important to not concatinate the results as strings,
    // as doing so will break some characters (like some emojis) that are made of multiple tokens.
    // by using an array of tokens, we can decode them correctly together.
    const resString: string = context.decode(Uint32Array.from(res));
    
    const lastPart = resString.split("ASSISTANT:").reverse()[0];
    if (lastPart.includes("USER:"))
        break;
}

const a1 = context.decode(Uint32Array.from(res)).split("USER:")[0];
console.log("AI: " + a1);
```

#### With grammar
Use this to direct the model to generate a specific format of text, like `JSON` for example.

> **Note:** there's an issue with some grammars where the model won't stop generating output,
> so it's advised to use it together with `maxTokens` set to the context size of the model

```typescript
import {fileURLToPath} from "url";
import path from "path";
import {LlamaModel, LlamaGrammar, LlamaContext, LlamaChatSession} from "node-llama-cpp";

const __dirname = path.dirname(fileURLToPath(import.meta.url));

const model = new LlamaModel({
    modelPath: path.join(__dirname, "models", "codellama-13b.Q3_K_M.gguf")
})
const grammar = await LlamaGrammar.getFor("json");
const context = new LlamaContext({
    model,
    grammar
});
const session = new LlamaChatSession({context});


const q1 = 'Create a JSON that contains a message saying "hi there"';
console.log("User: " + q1);

const a1 = await session.prompt(q1, {maxTokens: context.getContextSize()});
console.log("AI: " + a1);
console.log(JSON.parse(a1));


const q2 = 'Add another field to the JSON with the key being "author" and the value being "LLama"';
console.log("User: " + q2);

const a2 = await session.prompt(q2, {maxTokens: context.getContextSize()});
console.log("AI: " + a2);
console.log(JSON.parse(a2));
```

### CLI
```
Usage: node-llama-cpp <command> [options]

Commands:
  node-llama-cpp download      Download a release of llama.cpp and compile it
  node-llama-cpp build         Compile the currently downloaded llama.cpp
  node-llama-cpp clear [type]  Clear files created by node-llama-cpp
  node-llama-cpp chat          Chat with a LLama model

Options:
  -h, --help     Show help                                                                 [boolean]
  -v, --version  Show version number                                                       [boolean]
```

#### `download` command
```
node-llama-cpp download

Download a release of llama.cpp and compile it

Options:
  -h, --help             Show help                                                         [boolean]
      --repo             The GitHub repository to download a release of llama.cpp from. Can also be
                         set via the NODE_LLAMA_CPP_REPO environment variable
                                                           [string] [default: "ggerganov/llama.cpp"]
      --release          The tag of the llama.cpp release to download. Set to "latest" to download t
                         he latest release. Can also be set via the NODE_LLAMA_CPP_REPO_RELEASE envi
                         ronment variable                               [string] [default: "latest"]
  -a, --arch             The architecture to compile llama.cpp for                          [string]
  -t, --nodeTarget       The Node.js version to compile llama.cpp for. Example: v18.0.0     [string]
      --metal            Compile llama.cpp with Metal support. Can also be set via the NODE_LLAMA_CP
                         P_METAL environment variable                     [boolean] [default: false]
      --cuda             Compile llama.cpp with CUDA support. Can also be set via the NODE_LLAMA_CPP
                         _CUDA environment variable                       [boolean] [default: false]
      --skipBuild, --sb  Skip building llama.cpp after downloading it     [boolean] [default: false]
  -v, --version          Show version number                                               [boolean]
```

#### `build` command
```
node-llama-cpp build

Compile the currently downloaded llama.cpp

Options:
  -h, --help        Show help                                                              [boolean]
  -a, --arch        The architecture to compile llama.cpp for                               [string]
  -t, --nodeTarget  The Node.js version to compile llama.cpp for. Example: v18.0.0          [string]
      --metal       Compile llama.cpp with Metal support. Can also be set via the NODE_LLAMA_CPP_MET
                    AL environment variable                               [boolean] [default: false]
      --cuda        Compile llama.cpp with CUDA support. Can also be set via the NODE_LLAMA_CPP_CUDA
                     environment variable                                 [boolean] [default: false]
  -v, --version     Show version number                                                    [boolean]
```

#### `clear` command
```
node-llama-cpp clear [type]

Clear files created by node-llama-cpp

Options:
  -h, --help     Show help                                                                 [boolean]
      --type     Files to clear        [string] [choices: "source", "build", "all"] [default: "all"]
  -v, --version  Show version number                                                       [boolean]
```

#### `chat` command
```
node-llama-cpp chat

Chat with a LLama model

Required:
  -m, --model  LLama model file to use for the chat                              [string] [required]

Optional:
  -i, --systemInfo       Print llama.cpp system info                      [boolean] [default: false]
  -s, --systemPrompt     System prompt to use against the model. [default value: You are a helpful,
                         respectful and honest assistant. Always answer as helpfully as possible. If
                          a question does not make any sense, or is not factually coherent, explain
                         why instead of answering something not correct. If you don't know the answe
                         r to a question, please don't share false information.]
  [string] [default: "You are a helpful, respectful and honest assistant. Always answer as helpfully
                                                                                        as possible.
                If a question does not make any sense, or is not factually coherent, explain why ins
   tead of answering something not correct. If you don't know the answer to a question, please don't
                                                                          share false information."]
  -w, --wrapper          Chat wrapper to use. Use `auto` to automatically select a wrapper based on
                         the model's BOS token
                   [string] [choices: "auto", "general", "llamaChat", "chatML"] [default: "general"]
  -c, --contextSize      Context size to use for the model                  [number] [default: 4096]
  -g, --grammar          Restrict the model response to a specific grammar, like JSON for example
     [string] [choices: "text", "json", "list", "arithmetic", "japanese", "chess"] [default: "text"]
  -t, --temperature      Temperature is a hyperparameter that controls the randomness of the generat
                         ed text. It affects the probability distribution of the model's output toke
                         ns. A higher temperature (e.g., 1.5) makes the output more random and creat
                         ive, while a lower temperature (e.g., 0.5) makes the output more focused, d
                         eterministic, and conservative. The suggested temperature is 0.8, which pro
                         vides a balance between randomness and determinism. At the extreme, a tempe
                         rature of 0 will always pick the most likely next token, leading to identic
                         al outputs in each run. Set to `0` to disable.        [number] [default: 0]
  -k, --topK             Limits the model to consider only the K most likely next tokens for samplin
                         g at each step of sequence generation. An integer number between `1` and th
                         e size of the vocabulary. Set to `0` to disable (which uses the full vocabu
                         lary). Only relevant when `temperature` is set to a value greater than 0.
                                                                              [number] [default: 40]
  -p, --topP             Dynamically selects the smallest set of tokens whose cumulative probability
                          exceeds the threshold P, and samples the next token only from this set. A
                         float number between `0` and `1`. Set to `1` to disable. Only relevant when
                          `temperature` is set to a value greater than `0`. [number] [default: 0.95]
      --maxTokens, --mt  Maximum number of tokens to generate in responses. Set to `0` to disable. S
                         et to `-1` to set to the context size                 [number] [default: 0]

Options:
  -h, --help     Show help                                                                 [boolean]
  -v, --version  Show version number                                                       [boolean]
```
