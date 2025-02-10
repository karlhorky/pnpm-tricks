# pnpm Tricks

A collection of useful tricks for pnpm

## Fail `pnpm install` on pnpm v10 ignored build scripts

[pnpm v10 blocks lifecycle scripts by default](https://socket.dev/blog/pnpm-10-0-0-blocks-lifecycle-scripts-by-default), which means that dependencies with build scripts in lifecycle scripts such as [`esbuild`](https://www.npmjs.com/package/esbuild), [`sharp`](https://www.npmjs.com/package/sharp) and [`bcrypt`](https://www.npmjs.com/package/bcrypt) will no longer be built by default - instead they display a `Ignored build scripts:` warning at the bottom of the `pnpm install` output:

```bash
$ pnpm install
Lockfile is up to date, resolution step is skipped
Progress: resolved 1, reused 0, downloaded 0, added 0
Packages: +565
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Progress: resolved 565, reused 565, downloaded 0, added 363
Progress: resolved 565, reused 565, downloaded 0, added 565, done

dependencies:
+ react 19.0.0
+ react-dom 19.0.0

devDependencies:
+ @crxjs/vite-plugin 2.0.0-beta.31
+ @next/eslint-plugin-next 15.1.6
+ @types/chrome 0.0.301
+ @types/node 22.13.1
+ @types/react 19.0.8
+ @types/react-dom 19.0.3
+ @typescript-eslint/utils 8.23.0
+ @vitejs/plugin-react 4.3.4
+ eslint-config-upleveled 9.0.0
+ globals 15.14.0
+ prettier 3.4.2
+ typescript 5.7.3
+ vite 6.1.0

Ignored build scripts: esbuild. Run "pnpm approve-builds" to pick which dependencies should be allowed to run scripts.

Done in 2s
```

This warning is subtle and can be easily missed. Also, the exit code of `pnpm install` is `0`, so ignored build scripts will not fail workflows on CI:

```bash
$ pnpm install

...

Ignored build scripts: esbuild. Run "pnpm approve-builds" to pick which dependencies should be allowed to run scripts.

Done in 2s

$ echo $?
0
```

To always fail with exit code `1` on pnpm v10 ignored build scripts, loop over each line in the `pnpm install` output, set a variable to `true` if `Ignored build scripts:` message is found and, if the variable is not set to `false`, set the exit code to `1`:

```bash
$ pnpm install | { has_ignored_build_scripts=false; while IFS= read -r line; do echo "$line"; [[ "$line" == *"Ignored build scripts:"* ]] && has_ignored_build_scripts=true; done; [[ "$has_ignored_build_scripts" = false ]]; }

...

Ignored build scripts: esbuild. Run "pnpm approve-builds" to pick which dependencies should be allowed to run scripts.

Done in 2s

$ echo $?
1
```

On GitHub Actions:

`.github/workflows/ci.yml`

```yml
      - name: Install dependencies, failing on ignored build scripts
        # Fail on pnpm v10 ignored build scripts
        run: pnpm install | { has_ignored_build_scripts=false; while IFS= read -r line; do echo "$line"; [[ "$line" == *"Ignored build scripts:"* ]] && has_ignored_build_scripts=true; done; [[ "$has_ignored_build_scripts" = false ]]; }

        # TODO: Switch to future `pnpm install` option to fail on pnpm v10 ignored build scripts, if accepted:
        # - https://github.com/pnpm/pnpm/issues/9032#issuecomment-2647428724
        # run: pnpm install --<flag here>
```

In case pnpm implements the option to fail on pnpm v10 ignored build scripts, then switching to this option would be simpler:

- https://github.com/pnpm/pnpm/issues/9032#issuecomment-2647428724
