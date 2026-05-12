name: Morphed YouTube Universal Mirror APK

on:
  workflow_dispatch:
  schedule:
    - cron: "0 * * * *"

permissions:
  contents: write

env:
  APP_DISPLAY: YouTube Universal
  APP_SLUG: youtube-universal-morphe

jobs:
  check-version:
    runs-on: ubuntu-latest

    outputs:
      should_build: ${{ steps.compare.outputs.should_build }}
      asset_name: ${{ steps.pick.outputs.asset_name }}
      asset_url: ${{ steps.pick.outputs.asset_url }}
      version: ${{ steps.pick.outputs.version }}
      tag: ${{ steps.pick.outputs.tag }}
      release_title: ${{ steps.pick.outputs.release_title }}
      versioned_name: ${{ steps.pick.outputs.versioned_name }}

    steps:
      - name: Fetch latest upstream release JSON
        run: |
          curl -fsSL \
            "https://api.github.com/repos/RookieEnough/Morphe-AutoBuilds/releases/latest" \
            -o release.json

      - name: Pick APK and extract version
        id: pick
        shell: bash
        run: |
          set -euo pipefail

          ASSET_NAME=$(jq -r --arg SLUG "$APP_SLUG" '
            .assets[]
            | select(.name | ascii_downcase | startswith($SLUG | ascii_downcase))
            | select(.name | test("\\.apk$"; "i"))
            | .name
          ' release.json | head -n 1)

          if [ -z "${ASSET_NAME:-}" ] || [ "$ASSET_NAME" = "null" ]; then
            echo "No matching APK found."
            jq -r '.assets[].name'
            exit 1
          fi

          ASSET_URL=$(jq -r --arg NAME "$ASSET_NAME" '
            .assets[]
            | select(.name == $NAME)
            | .browser_download_url
          ' release.json)

          VERSION=$(printf '%s\n' "$ASSET_NAME" | \
            sed -n 's/.*-v\([0-9][0-9.]*\)\.apk$/\1/p')

          if [ -z "${VERSION:-}" ]; then
            echo "Could not extract version."
            exit 1
          fi

          TAG="v${VERSION}"
          RELEASE_TITLE="${APP_DISPLAY} v${VERSION}"

          # Keep exact upstream filename
          VERSIONED_NAME="$ASSET_NAME"

          echo "asset_name=$ASSET_NAME" >> "$GITHUB_OUTPUT"
          echo "asset_url=$ASSET_URL" >> "$GITHUB_OUTPUT"
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
          echo "tag=$TAG" >> "$GITHUB_OUTPUT"
          echo "release_title=$RELEASE_TITLE" >> "$GITHUB_OUTPUT"
          echo "versioned_name=$VERSIONED_NAME" >> "$GITHUB_OUTPUT"

          echo "Selected asset: $ASSET_NAME"
          echo "Version: $VERSION"

      - name: Check if version already exists
        id: compare
        env:
          GH_TOKEN: ${{ github.token }}
        shell: bash
        run: |
          set -euo pipefail

          if gh release view "${{ steps.pick.outputs.tag }}" \
            -R "$GITHUB_REPOSITORY" >/dev/null 2>&1; then

            echo "Version already exists. Skipping build."
            echo "should_build=false" >> "$GITHUB_OUTPUT"
          else
            echo "New version detected."
            echo "should_build=true" >> "$GITHUB_OUTPUT"
          fi

  mirror:
    needs: check-version
    if: needs.check-version.outputs.should_build == 'true'
    runs-on: ubuntu-latest

    steps:
      - name: Download APK
        run: |
          curl -fL "${{ needs.check-version.outputs.asset_url }}" \
            -o "${{ needs.check-version.outputs.versioned_name }}"

          ls -lh "${{ needs.check-version.outputs.versioned_name }}"

      - name: Create release
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          gh release create "${{ needs.check-version.outputs.tag }}" \
            "${{ needs.check-version.outputs.versioned_name }}" \
            -R "$GITHUB_REPOSITORY" \
            --title "${{ needs.check-version.outputs.release_title }}" \
            --notes "## YouTube Universal Morphe Mirror

          Mirrored from \`RookieEnough/Morphe-AutoBuilds\`

          - Version: \`${{ needs.check-version.outputs.version }}\`
          - Original asset: \`${{ needs.check-version.outputs.asset_name }}\`
          - Mirrored asset: \`${{ needs.check-version.outputs.versioned_name }}\`

          This workflow only runs the mirror job when the APK version changes." \
            --latest
