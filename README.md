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
perl -i -pe '$exists ||= /^minimum-release-age-exclude\[\]=webpack$/; $_ .= "minimum-release-age-exclude[]=webpack\n" if eof && !$exists' "$LOCALAPPDATA/pnpm/config/rc"

# macOS
perl -i -pe '$exists ||= /^minimum-release-age-exclude\[\]=webpack$/; $_ .= "minimum-release-age-exclude[]=webpack\n" if eof && !$exists' "$HOME/Library/Preferences/pnpm/rc"

# Linux
perl -i -pe '$exists ||= /^minimum-release-age-exclude\[\]=webpack$/; $_ .= "minimum-release-age-exclude[]=webpack\n" if eof && !$exists' "$HOME/.config/pnpm/rc"
```

Reproduction: https://github.com/karlhorky/repro-pnpm-minimumReleaseAgeExclude-global-cross-platform

## Convert `patch-package` patches to pnpm patches

To convert existing `patch-package` patches in `patches/` to versionless pnpm patches in the same directory, try the following script:

`scripts/patch-package_to_pnpm.sh`

```bash
#!/usr/bin/env bash

set -o errexit
set -o nounset
set -o pipefail

root_dir=$(pwd)
patches_dir="${root_dir}/patches"
store_dir="${root_dir}/node_modules/.pnpm"

# Required tools
for cmd in git rsync node; do
  if ! command -v "${cmd}" >/dev/null 2>&1; then
    echo "Error: required command '${cmd}' not found on PATH" >&2
    exit 1
  fi
done

mapfile -t patch_files < <(find "${patches_dir}" -maxdepth 1 -type f -name '*+*.patch' | sort)

if [[ ${#patch_files[@]} -eq 0 ]]; then
  echo "No patch-package patches found in ${patches_dir}."
  exit 0
fi

for patch_file in "${patch_files[@]}"; do
  base=$(basename "${patch_file}" .patch)
  version="${base##*+}"
  encoded="${base%+${version}}"
  if [[ -z "${encoded}" || -z "${version}" ]]; then
    echo "Skipping ${patch_file}: unable to parse name." >&2
    continue
  fi

  target_encoded="${encoded}"
  if [[ "${target_encoded}" == *"++"* ]]; then
    target_encoded="${target_encoded##*++}"
  fi

  target_package="${target_encoded//+/\/}"

  echo "Converting ${patch_file} -> ${target_package}"

  target_dir=""

  # 1) Prefer the real path behind node_modules/<pkg> if available
  if [[ -e "${root_dir}/node_modules/${target_package}" ]]; then
    if command -v realpath >/dev/null 2>&1; then
      rp="$(realpath "${root_dir}/node_modules/${target_package}" 2>/dev/null || true)"
    else
      rp=""
    fi
    if [[ -n "${rp}" && -f "${rp}/package.json" ]]; then
      target_dir="${rp}"
    fi
  fi

  # 2) Exact match inside the virtual store
  if [[ -z "${target_dir}" && -d "${store_dir}" ]]; then
    candidate=$(find "${store_dir}" -maxdepth 1 -type d -name "${target_encoded}@${version}*" | head -n 1 || true)
    if [[ -n "${candidate}" && -d "${candidate}/node_modules/${target_package}" ]]; then
      target_dir="${candidate}/node_modules/${target_package}"
    fi
  fi

  # 3) Any installed version in the virtual store
  if [[ -z "${target_dir}" && -d "${store_dir}" ]]; then
    candidate=$(find "${store_dir}" -maxdepth 1 -type d -name "${target_encoded}@*" | head -n 1 || true)
    if [[ -n "${candidate}" && -d "${candidate}/node_modules/${target_package}" ]]; then
      target_dir="${candidate}/node_modules/${target_package}"
    fi
  fi

  # 4) Last resort: scan workspace for package.json
  if [[ -z "${target_dir}" ]]; then
    while IFS= read -r pkg_json; do
      name=$(node --input-type=module -e 'import fs from "node:fs"; const p=process.argv[1]; try{const d=JSON.parse(fs.readFileSync(p,"utf8")); process.stdout.write(d.name||"");}catch{}' "${pkg_json}" 2>/dev/null || true)
      if [[ "${name}" == "${target_package}" ]]; then
        target_dir=$(dirname "${pkg_json}")
        break
      fi
    done < <(find "${root_dir}" -maxdepth 4 -name package.json 2>/dev/null)
  fi

  if [[ -z "${target_dir}" ]]; then
    echo "  Unable to locate ${target_package} (filename version ${version}); skipping." >&2
    continue
  fi

  tmp_dir=$(mktemp -d "${root_dir}/.tmp-pnpm-patch.XXXXXX")
  rsync --archive --delete "${target_dir}/" "${tmp_dir}/"

  (cd "${tmp_dir}" && git init --initial-branch=main >/dev/null)
  (cd "${tmp_dir}" && git config user.email "patch@upleveled.io" >/dev/null)
  (cd "${tmp_dir}" && git config user.name "pnpm patch converter" >/dev/null)
  (cd "${tmp_dir}" && git add --all >/dev/null)
  (cd "${tmp_dir}" && git commit --message "baseline" >/dev/null)

  applied=0
  for strip in 0 1 2 3 4 5 6 7; do
    if (cd "${tmp_dir}" && git apply --check -p"${strip}" "${patch_file}" >/dev/null 2>&1); then
      (cd "${tmp_dir}" && git apply -p"${strip}" "${patch_file}")
      applied=1
      break
    fi
    if (cd "${tmp_dir}" && git apply --reverse --check -p"${strip}" "${patch_file}" >/dev/null 2>&1); then
      (cd "${tmp_dir}" && git apply --reverse -p"${strip}" "${patch_file}")
      (cd "${tmp_dir}" && git add --all >/dev/null)
      (cd "${tmp_dir}" && git commit --amend --no-edit >/dev/null)
      if (cd "${tmp_dir}" && git apply --check -p"${strip}" "${patch_file}" >/dev/null 2>&1); then
        (cd "${tmp_dir}" && git apply -p"${strip}" "${patch_file}")
        applied=1
        break
      fi
    fi
  done

  if [[ "${applied}" -eq 0 ]]; then
    echo "  Failed to apply ${patch_file}" >&2
    rm -rf "${tmp_dir}"
    continue
  fi

  diff_output=$(cd "${tmp_dir}" && git diff)
  if [[ -z "${diff_output}" ]]; then
    echo "  ${patch_file} produced no changes; skipping output." >&2
    rm -rf "${tmp_dir}"
    continue
  fi

  sanitized="${target_encoded//+/__}"
  new_patch_path="patches/${sanitized}.patch"
  printf '%s' "${diff_output}" > "${root_dir}/${new_patch_path}"

  node --input-type=module - "${root_dir}/package.json" "${target_package}" "${new_patch_path}" <<'NODE'
import fs from 'node:fs';
const [pkgPath, dep, patchPath] = process.argv.slice(2);
const data = JSON.parse(fs.readFileSync(pkgPath, 'utf8'));
data.pnpm ??= {};
data.pnpm.patchedDependencies ??= {};
for (const k of Object.keys(data.pnpm.patchedDependencies)) {
  if (k === dep || k.startsWith(`${dep}@`)) delete data.pnpm.patchedDependencies[k];
}
data.pnpm.patchedDependencies[dep] = patchPath;
fs.writeFileSync(pkgPath, JSON.stringify(data, null, 2) + '\n');
NODE

  rm -rf "${tmp_dir}"
done

echo "All patch-package patches processed (versionless). Run 'pnpm install' to apply."
```

Make the script executable and run it from the project root:

```bash
chmod +x scripts/patch-package_to_pnpm.sh
./scripts/patch-package_to_pnpm.sh
```

Example output:

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
