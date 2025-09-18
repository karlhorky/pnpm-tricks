# pnpm Tricks

A collection of useful tricks for pnpm

## Configure `minimumReleaseAge` and `minimumReleaseAgeExclude` globally

Caveat: @zkochan [suggests](https://github.com/pnpm/pnpm/issues/9921#issuecomment-3292521911) considering [Config Dependencies](https://pnpm.io/config-dependencies) instead of global configuration

To configure pnpm [`minimumReleaseAge`](https://pnpm.io/settings#minimumreleaseage) globally, use `pnpm config set` commands:

```bash
# minimumReleaseAge of 7 days
pnpm config set minimumReleaseAge 10080 --global
```

[`minimumReleaseAgeExclude`](https://pnpm.io/settings#minimumreleaseageexclude) is a bit more complicated to configure globally, because pnpm does not allow setting complex values like arrays using `pnpm config set`. In this case, use Perl commands with platform-specific global configuration file paths to set `minimumReleaseAgeExclude` globally (these work only once the global config files exist):

```bash
# Windows
perl -i -pe '$exists ||= /^minimum-release-age-exclude\[\]=webpack\r?$/; $_ .= "minimum-release-age-exclude[]=webpack\n" if eof && !$exists' "$LOCALAPPDATA/pnpm/config/rc"

# macOS
perl -i -pe '$exists ||= /^minimum-release-age-exclude\[\]=webpack\r?$/; $_ .= "minimum-release-age-exclude[]=webpack\n" if eof && !$exists' "$HOME/Library/Preferences/pnpm/rc"

# Linux
perl -i -pe '$exists ||= /^minimum-release-age-exclude\[\]=webpack\r?$/; $_ .= "minimum-release-age-exclude[]=webpack\n" if eof && !$exists' "$HOME/.config/pnpm/rc"
```

Reproduction: https://github.com/karlhorky/repro-pnpm-minimumReleaseAgeExclude-global-cross-platform

## Fail `pnpm install` on pnpm v10 ignored build scripts

**Update:** Failing on ignored build scripts is now built into pnpm! 🎉 Use [the `strict-dep-builds` setting](https://github.com/pnpm/pnpm/pull/9071#issuecomment-2650192097) (default `false`) introduced in [pnpm v10.3.0](https://github.com/pnpm/pnpm/releases/tag/v10.3.0)

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

To always fail with exit code `1` on pnpm v10 ignored build scripts, loop over each line in the `pnpm install` output to look for the `Ignored build scripts:` message. If the message is found, set a variable to `true` and exit with code `1`:

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
        shell: bash # Use Git Bash on Windows for command group above

        # TODO: Switch to future `pnpm install` option to fail on pnpm v10 ignored build scripts, if accepted:
        # - https://github.com/pnpm/pnpm/issues/9032#issuecomment-2647428724
        # run: pnpm install --<flag here>
```

In case pnpm implements the option to fail on pnpm v10 ignored build scripts, then switching to this option would be simpler:

- https://github.com/pnpm/pnpm/issues/9032#issuecomment-2647428724
