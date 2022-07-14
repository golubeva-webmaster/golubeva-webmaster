# Создание Nuxt модуля
> #### Дисклаймер
> Эта статья является описанием собственного опыта создания модуля накст, с компонентами и плагинами. И является дополнением к документации и статье:
> - https://nuxtjs.org/docs/internals-glossary/internals/
> - https://nuxtjs.org/docs/directory-structure/modules/
> - https://medium.com/carepenny/creating-a-nuxt-module-1c6e3cdf1037
> В отличии от указанных источников, у меня все примеры на TypoScript
{.is-info}



## Регистрация модуля
Название: commonModule
Точка входа: modules/commonModule/index.ts

### Глобальные параметры
Расскажем nuxt о новом модуле и пропишем глобальные параметры в nuxt.config.ts:
```ts
modules: [
   '@nuxtjs/i18n',
   '@nuxtjs/axios',
   ['~/modules/commonModule', { namespace: 'commonModule', option1: 'something' }] //* Попадает в переменную moduleOptions в точке входа модуля
 ],

 commonModule: { nuxtInitialValue: 22, debug: true }, //* В точке входа в модуль это будет: nuxt.options.commonModule
```


- Объект *{ namespace: 'commonModule', option1: 'something' }* - попадает в переменную moduleOptions в точке входа модуля.
- *commonModule* - попадает в nuxt.options.commonModule.
Это два способа описать глобальные опции модуля.

**Доступ к параметрам из точки входа модуля**

В точке входа `modules/commonModule/index.ts` обращаемся к глобально описанным опциям так:

```ts
const commonModule: Module<Options> = function (moduleOptions) {

  const { nuxt } = this

  const options = {
    ...moduleOptions,             // { namespace: 'commonModule', option1: 'something' }
    ...nuxt.options.commonModule  // { nuxtInitialValue: 22, debug: true }
  }
```

**Доступ к параметрам из плагина**

Доступ к глобальным опциям nuxt модуля, определенным в nuxt.config.ts из плагина helpers
`modules/commonModule/plugins/helpers/helpers.ts`:
```ts
const options = JSON.parse(`<%= JSON.stringify(options) %>`) //! Доступ к глобальным опциям nuxt модуля, определенным в nuxt.config.ts
const { namespace } = options
console.log(`===== helpers plugin ===== ${namespace} options: `, options)
```

## Плагины

Например, создадим плагин helpers со списком функций, которые хотим использовать как в модуле commonModule, так и в любом месте приложения.
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

**Способы сделать inject**

Далее необходимо сделать inject функций плагина, чтобы иметь возможность использовать их.
В статье, указанной в начале, inject советуют делать [как в файле](https://medium.com/carepenny/creating-a-nuxt-module-1c6e3cdf1037#2264)
`modules/commonModule/plugins/index.ts`, предварительно импортировав функции плагинов из `modules/commonModule/plugins/helpers/index.ts`

[Так и в файлах каждого плагина](https://medium.com/carepenny/creating-a-nuxt-module-1c6e3cdf1037#b476)
То есть в:
`modules/commonModule/plugins/helpers/helpers.ts`
`...`

Мне больше понравилась вторая идея. Т.к. в итоге, после создания модуля, сохраняется прозрачность логики. Получается:
- Иннъекции в каждом плагине свои
- Собираем все плагины воедино в точке входа модуля
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

У меня получилось так
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

Таким образом, в результате многих часов чтения статей и документации, проб и хождения по граблям, я пришла к выводу, что следующие файлы, которые описывает Jamie Curnow в статье, мне не нужны:
- `modules/commonModule/plugins/index.ts`
- `modules/commonModule/plugins/helpers/index.ts`

Но это встеки дело вкуса каждого. Оба подхода имеют право на существование. Я - за оправданный минимализм.

**Вызов плагина**

Вот так вызываем функции плагинов из любого места. Будь то собственный модуль, в котором определен плагин. Или любое место в приложении.
```ts
this.$nuxt.$options.$capitalize(‘test’)
```

Определенный таким образом плагин `modules/commonModule/plugins/helpers/helpers.ts` после перебилдинга приложения “прорастает” в `.nuxt/commonModule/plugins/helpers/helpers.ts`

## Компоненты

Я не буду переписывать описание создания компонентов, оно отлично описано в указанных источниках. Ниже моменты, которые помогли мне.

Без этого блока нельзя будет вставлять компоненты нашего модуля на страницы *.vue
Чтобы в TypoScript можно было импортировать .vue файлы. Например так:
`modules/commonModule/components/lib/index.ts`
```ts
import WebcamCard from './WebcamCard.vue'
import FileManager from './FileManager.vue'
```

Потребуется прописать декларацию. Создать в корне приложения файл fileName.d.ts
`module.d.ts:`
```ts
declare module "*.vue" {
 import Vue from 'vue'
 export default Vue
}
```

**Укажем в конфигурации nuxt откуда брать компоненты**

В `nuxt.config.ts` указать папку, откуда брать модули. Теперь у нас это не только папка components, как было до создания нами собственного модуля, но и `modules/commonModule/components/lib`.
То есть в `nuxt.config.ts` было так:
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
О свойстве components можно [почитать тут](https://nuxtjs.org/docs/configuration-glossary/configuration-components/)

Проверить, видит ли наш проект наш commonModule вместе с его компонентами можно так
Пересобираем проект: **npm run dev**
И проверяем файл со списком nuxt компонентов: `.nuxt/components/readme.md` и видим, что наш компонент, например под названием FileManager (`modules/commonModule/components/lib/FileManager.vue`) доступен для вставки на страницы как
```html
<FileManager>
<file-manager>
```
и читается из `modules/commonModule/components/lib/FileManager.vue`

```md
- `<FileManager>` | `<file-manager>` (modules/commonModule/components/lib/FileManager.vue)
```








