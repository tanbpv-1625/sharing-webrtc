# Vue 3 + TypeScript + Vite

This template should help get you started developing with Vue 3 and TypeScript in Vite. The template uses Vue 3 `<script setup>` SFCs, check out the [script setup docs](https://v3.vuejs.org/api/sfc-script-setup.html#sfc-script-setup) to learn more.

## Recommended IDE Setup

- [VS Code](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar)

## Type Support For `.vue` Imports in TS

Since TypeScript cannot handle type information for `.vue` imports, they are shimmed to be a generic Vue component type by default. In most cases this is fine if you don't really care about component prop types outside of templates. However, if you wish to get actual prop types in `.vue` imports (for example to get props validation when using manual `h(...)` calls), you can enable Volar's Take Over mode by following these steps:

1. Run `Extensions: Show Built-in Extensions` from VS Code's command palette, look for `TypeScript and JavaScript Language Features`, then right click and select `Disable (Workspace)`. By default, Take Over mode will enable itself if the default TypeScript extension is disabled.
2. Reload the VS Code window by running `Developer: Reload Window` from the command palette.

You can learn more about Take Over mode [here](https://github.com/johnsoncodehk/volar/discussions/471)

# Firebase signling

1. Go to https://console.firebase.google.com/ => Add project

2. Add Firebase to your web app: Open a new project => Select web

<img width="1439" alt="Screen Shot 2022-10-19 at 15 35 12" src="https://user-images.githubusercontent.com/51943633/196640166-ca740fa8-940e-4ef8-b1f4-4e9dde288ac0.png">

3. Set up Cloud Storage: Select storage => select "Start in test mode" => Next => Select Cloud Storage location => Done

<img width="1436" alt="Screen Shot 2022-10-19 at 15 40 35" src="https://user-images.githubusercontent.com/51943633/196641392-03e95202-3e99-40d6-a69f-6d70c0acda56.png">

4. Select Project settings => copy firebaseConfig to your app.

5. Enable API https://firebase.google.com/docs/projects/api/workflow_set-up-and-manage-project#enable-api

6. Go to tab Indexes => Add index 'answerCandidates', 'offerCandidates', query scope 'Collection'.
