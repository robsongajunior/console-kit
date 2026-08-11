# Nuxt console-kit
Nós vamos migrar o projeto ![azion-console-kit](https://github.com/aziontech/azion-console-kit) para esse novo monorepo.


## PROJECT PATHS
- /Users/unknown1/www/azion/azion-console-kit/
- /Users/unknown1/www/azion/webkit/packages/webkit
  - @aziontech/webkit at NPM
- /Users/unknown1/www/azion/webkit/packages/theme
  - @aziontech/theme at NPM
- /Users/unknown1/www/azion/webkit/packages/icons
  - @aziontech/icons at NPM


## REPOSITORY ARCHITECTURE

Como monorepo estruturado penso em algo como:

```
console-kit/
├── apps/
│   ├── storybook/   # Component docs and development playground
│   └── console/     # Interactive icon browser
├── packages/
│   ├── components/     # Reusable platform components
│   ├── services/       # Reusable API/SDK Abstraction
├── package.json        # Root workspace scripts
└── pnpm-workspace.yaml
```

## Tecnologies

### general

**devDependencies:**
- commitlint
  - @commitlint/cli
  - @commitlint/config-conventional
- eslint
  - vue-eslint-parser
  - eslint-plugin-xss
  - eslint-plugin-vue
  - eslint-plugin-security
  - eslint-plugin-no-unsanitized
- zod
- prettier
- husky
- vitest



### apps/storybook
- storybook - latest 

### apps/console

#### Front-End

**dependencies:**
- Nuxt
- @aziontech/webkit - 4.3.0+
- @aziontech/theme - 4.3.0+
- @aziontech/icons - 4+
- Tailwind - 4+
- JSONFORM
  - @jsonforms/core - 3.8.0
  - @jsonforms/vue - 3.8.0
  - @jsonforms/vue-vanilla - 3.8.0
- motino-v - ˆ2.3.0

#### Monitoring

**dependencies:**
- Sentry
  - @sentry/vue
  - verificar se tem algo para Nuxt 
