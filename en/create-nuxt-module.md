# Create Nuxt module
> #### Disclaimer
> This article is a description of my own experience of creating a naxt module, with components and plugins. And is an addition to the documentation and the article:
> - https://nuxtjs.org/docs/internals-glossary/internals/
> - https://nuxtjs.org/docs/directory-structure/modules/
> - https://medium.com/carepenny/creating-a-nuxt-module-1c6e3cdf1037
> Unlike the indicated sources, I have all the examples in TypoScript
{.is-info}



## Module registration
Name: commonModule
Entry point: modules/commonModule/index.ts

### Global module options
Let's tell nuxt about the new module and set the global options in nuxt.config.ts:
```ts
modules: [
   '@nuxtjs/i18n',
   '@nuxtjs/axios',
   ['~/modules/commonModule', { namespace: 'commonModule', option1: 'something' }] //* Попадает в переменную moduleOptions в точке входа модуля
 ],

 commonModule: { nuxtInitialValue: 22, debug: true }, //* В точке входа в модуль это будет: nuxt.options.commonModule
```


- Object *{ namespace: 'commonModule', option1: 'something' }* - gets into the moduleOptions variable at the module's entry point.
- *commonModule* - hits в nuxt.options.commonModule.
These are two ways to describe the global options of a module.

**Accessing parameters from a module Entry point**

In the entry point `modules/commonModule/index.ts` we access the globally defined options like this:

```ts
const commonModule: Module<Options> = function (moduleOptions) {

  const { nuxt } = this

  const options = {
    ...moduleOptions,             // { namespace: 'commonModule', option1: 'something' }
    ...nuxt.options.commonModule  // { nuxtInitialValue: 22, debug: true }
  }
```

**Accessing parameters from plugin**

Access global nuxt module options defined in nuxt.config.ts from helpers plugin
`modules/commonModule/plugins/helpers/helpers.ts`:
```ts
const options = JSON.parse(`<%= JSON.stringify(options) %>`) //! Доступ к глобальным опциям nuxt модуля, определенным в nuxt.config.ts
const { namespace } = options
console.log(`===== helpers plugin ===== ${namespace} options: `, options)
```

## Plugins

For example, let's create a helpers plugin with a list of functions that we want to use both in the commonModule module and anywhere in the application.
`modules/commonModule/plugins/helpers/helpers.ts:`
```ts
import { Plugin } from '@nuxt/types'
import { FileStateFile, IHelpers } from './interfaces';

const helperFunctions: IHelpers = {

  capitalize: (str: string): string => {
    return str.charAt(0).toUpperCase() + str.slice(1)
  },
  ...
}
```

**Ways to inject**

Next, you need to inject the plugin functions in order to be able to use them.
In the article mentioned at the beginning, inject is advised to do [as in the file] (https://medium.com/carepenny/creating-a-nuxt-module-1c6e3cdf1037#2264)
`modules/commonModule/plugins/index.ts` after importing plugin functions from `modules/commonModule/plugins/helpers/index.ts`

[So in the files of each plugin](https://medium.com/carepenny/creating-a-nuxt-module-1c6e3cdf1037#b476)
То есть в:
`modules/commonModule/plugins/helpers/helpers.ts`
`...`

I liked the second idea better. Because as a result, after the creation of the module, the transparency of the logic is preserved. It turns out:
- Injections in each plugin are different
- We collect all the plugins together at the module entry point
`modules/commonModule/index.ts` и регистрируем плагины в модуле (addPlugin)
```ts
  const pluginsToSync = [
    'plugins/helpers/helpers.ts',
    ...
  ]

  for (const pathString of pluginsToSync) {
    this.addPlugin({
      src: resolve(__dirname, pathString),
      fileName: join(namespace, pathString),
      options
    })
  }
```

I got it like this
`modules/commonModule/plugins/helpers/helpers.ts:`
```ts
import { Plugin } from '@nuxt/types'
import { FileStateFile, IHelpers } from './interfaces';

const helperFunctions: IHelpers = {

  capitalize: (str: string): string => {
    return str.charAt(0).toUpperCase() + str.slice(1)
  },
  ...
}

const helpersPlugin: Plugin = (context, inject) => {
  inject('capitalize', helperFunctions.capitalize)
  ...
}

export default helpersPlugin;
```

Thus, as a result of many hours of reading articles and documentation, trial and error, I came to the conclusion that I do not need the following files, which Jamie Curnow describes in the article:- `modules/commonModule/plugins/index.ts`
- `modules/commonModule/plugins/helpers/index.ts`

But it's all a matter of taste for everyone. Both approaches have a right to exist. I am for justified minimalism.

**Plugin call**

This is how we call plugin functions from anywhere. Whether it's a custom module where the plugin is defined. Or anywhere in the app.
```ts
this.$nuxt.$options.$capitalize(‘test’)
```

The plugin defined in this way `modules/commonModule/plugins/helpers/helpers.ts` after rebuilding the application "sprouts" into `.nuxt/commonModule/plugins/helpers/helpers.ts`

## Components

I will not rewrite the description of creating components, it is perfectly described in the indicated sources. Below are the points that helped me.

Without this block, it will not be possible to insert the components of our module into *.vue pages
So that TypoScript can import .vue files. For example like this:
`modules/commonModule/components/lib/index.ts`
```ts
import WebcamCard from './WebcamCard.vue'
import FileManager from './FileManager.vue'
```

You will need to write a declaration. Create a file fileName.d.ts in the root of the application
`module.d.ts:`
```ts
declare module "*.vue" {
 import Vue from 'vue'
 export default Vue
}
```

** Specify in the nuxt configuration where to get the components **

In `nuxt.config.ts` specify the folder where to get the modules from. Now we have not only the components folder, as it was before we created our own module, but also `modules/commonModule/components/lib`.
That is, in `nuxt.config.ts` it was like this:
```ts
components: true
```
Стало:
```ts
components: [
   '~/components',
   { path: 'modules/commonModule/components/lib', extensions: ['vue'] }
 ],
```
About the components property, you can [почитать тут](https://nuxtjs.org/docs/configuration-glossary/configuration-components/)

You can check if our project sees our commonModule along with its components like this
Rebuild project: **npm run dev**
And we check the file with the list of nuxt components: `.nuxt/components/readme.md` and see that our component, for example called FileManager (`modules/commonModule/components/lib/FileManager.vue`) is available for insertion into pages as
```html
<FileManager>
<file-manager>
```
and reading from `modules/commonModule/components/lib/FileManager.vue`

```md
- `<FileManager>` | `<file-manager>` (modules/commonModule/components/lib/FileManager.vue)
```








