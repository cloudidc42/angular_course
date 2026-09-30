# Part 75: Angular Schematics - สร้าง Custom `ng generate`

## Schematics คืออะไร

Schematics คือ code generation tools ที่ใช้กับ Angular CLI เช่น `ng generate component` ทำงานผ่าน Schematics

---

## 1. ติดตั้ง Schematics

```bash
npm install -g @angular-devkit/schematics-cli

# สร้าง schematics project
schematics blank --name=my-schematics

cd my-schematics
npm install
```

---

## 2. โครงสร้างไฟล์

```
my-schematics/
├── src/
│   ├── my-component/
│   │   ├── index.ts          ← factory function
│   │   ├── schema.json       ← input schema
│   │   └── files/            ← templates
│   │       ├── __name@dasherize__.component.ts.template
│   │       ├── __name@dasherize__.component.html.template
│   │       └── __name@dasherize__.component.scss.template
│   └── collection.json       ← schematics registry
├── package.json
└── tsconfig.json
```

---

## 3. สร้าง Custom Component Schematic

### schema.json

```json
{
  "$schema": "http://json-schema.org/schema",
  "$id": "MyComponentSchema",
  "title": "My Component Schema",
  "type": "object",
  "properties": {
    "name": {
      "type": "string",
      "description": "ชื่อ component",
      "$default": {
        "$source": "argv",
        "index": 0
      }
    },
    "path": {
      "type": "string",
      "description": "path ที่จะสร้าง component",
      "format": "path"
    },
    "module": {
      "type": "string",
      "description": "module ที่จะ declare component"
    },
    "style": {
      "type": "string",
      "default": "scss",
      "enum": ["css", "scss", "less", "none"],
      "description": "รูปแบบ stylesheet"
    },
    "hasStore": {
      "type": "boolean",
      "default": false,
      "description": "เพิ่ม NgRx store boilerplate"
    },
    "hasService": {
      "type": "boolean",
      "default": false,
      "description": "สร้าง service พร้อมกัน"
    }
  },
  "required": ["name"]
}
```

### index.ts - Factory Function

```typescript
// src/my-component/index.ts
import { 
  Rule, SchematicContext, Tree, chain,
  apply, url, applyTemplates, move,
  mergeWith, MergeStrategy
} from '@angular-devkit/schematics';
import { 
  strings, normalize, experimental 
} from '@angular-devkit/core';
import { 
  addDeclarationToModule,
  findModuleFromOptions
} from '@schematics/angular/utility/ng-ast-utils';

export interface MyComponentSchema {
  name: string;
  path?: string;
  module?: string;
  style: string;
  hasStore: boolean;
  hasService: boolean;
}

export function myComponent(options: MyComponentSchema): Rule {
  return (tree: Tree, context: SchematicContext) => {
    context.logger.info(`🚀 สร้าง component: ${options.name}`);

    return chain([
      createComponentFiles(options),
      options.hasService ? createServiceFile(options) : noop(),
      options.hasStore ? createStoreFiles(options) : noop(),
      updateModuleFile(options)
    ])(tree, context);
  };
}

function noop(): Rule {
  return (tree: Tree) => tree;
}

function createComponentFiles(options: MyComponentSchema): Rule {
  return (tree: Tree, context: SchematicContext) => {
    const templateSource = apply(url('./files/component'), [
      applyTemplates({
        ...strings,          // dasherize, classify, camelize, etc.
        ...options,
        name: options.name,
        selector: `app-${strings.dasherize(options.name)}`
      }),
      move(normalize(options.path || 'src/app'))
    ]);

    return mergeWith(templateSource, MergeStrategy.Default)(tree, context);
  };
}

function createServiceFile(options: MyComponentSchema): Rule {
  return (tree: Tree, context: SchematicContext) => {
    context.logger.info(`สร้าง service สำหรับ ${options.name}`);
    
    const templateSource = apply(url('./files/service'), [
      applyTemplates({
        ...strings,
        ...options
      }),
      move(normalize(options.path || 'src/app'))
    ]);

    return mergeWith(templateSource)(tree, context);
  };
}

function createStoreFiles(options: MyComponentSchema): Rule {
  return (tree: Tree, context: SchematicContext) => {
    context.logger.info(`สร้าง NgRx store สำหรับ ${options.name}`);
    
    const templateSource = apply(url('./files/store'), [
      applyTemplates({
        ...strings,
        ...options
      }),
      move(normalize(`${options.path || 'src/app'}/store/${strings.dasherize(options.name)}`))
    ]);

    return mergeWith(templateSource)(tree, context);
  };
}

function updateModuleFile(options: MyComponentSchema): Rule {
  return (tree: Tree, context: SchematicContext) => {
    if (!options.module) return tree;
    
    // หา module file
    const modulePath = options.module.endsWith('.module.ts') 
      ? options.module 
      : `${options.module}.module.ts`;
    
    if (!tree.exists(modulePath)) {
      context.logger.warn(`ไม่พบ module: ${modulePath}`);
      return tree;
    }

    const content = tree.read(modulePath)?.toString('utf-8') || '';
    const componentName = strings.classify(options.name) + 'Component';
    
    if (content.includes(componentName)) {
      context.logger.warn(`${componentName} มีใน module แล้ว`);
      return tree;
    }

    // เพิ่ม declaration
    const updatedContent = addComponentToModule(content, componentName, options.name);
    tree.overwrite(modulePath, updatedContent);
    
    return tree;
  };
}

function addComponentToModule(content: string, componentName: string, fileName: string): string {
  const importStatement = `import { ${componentName} } from './${strings.dasherize(fileName)}/${strings.dasherize(fileName)}.component';`;
  
  // เพิ่ม import statement
  let updated = content.replace(
    /(import\s+.*?\n)(\n*@NgModule)/,
    `$1${importStatement}\n$2`
  );
  
  // เพิ่มใน declarations
  updated = updated.replace(
    /declarations:\s*\[([\s\S]*?)\]/,
    (match, declarations) => {
      const trimmed = declarations.trim();
      if (!trimmed) return `declarations: [${componentName}]`;
      return `declarations: [${trimmed}, ${componentName}]`;
    }
  );
  
  return updated;
}
```

---

## 4. Template Files

### Component Template

```typescript
// src/my-component/files/component/__name@dasherize__.component.ts.template
import { Component, OnInit, ChangeDetectionStrategy } from '@angular/core';
<% if (hasService) { %>import { <%= classify(name) %>Service } from './<%= dasherize(name) %>.service';<% } %>
<% if (hasStore) { %>import { Store } from '@ngrx/store';<% } %>

@Component({
  selector: '<%= selector %>',
  templateUrl: './<%= dasherize(name) %>.component.html',
  styleUrls: ['./<%= dasherize(name) %>.component.<%= style %>'],
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class <%= classify(name) %>Component implements OnInit {
  <% if (hasService) { %>
  constructor(private <%= camelize(name) %>Service: <%= classify(name) %>Service) {}
  <% } else { %>
  constructor() {}
  <% } %>

  ngOnInit(): void {
    // TODO: Implement initialization
  }
}
```

### HTML Template

```html
<!-- src/my-component/files/component/__name@dasherize__.component.html.template -->
<div class="<%= dasherize(name) %>-container">
  <h2><%= classify(name) %></h2>
  <!-- TODO: Add your template here -->
</div>
```

### SCSS Template

```scss
// src/my-component/files/component/__name@dasherize__.component.scss.template
.<%= dasherize(name) %>-container {
  // TODO: Add your styles here
}
```

---

## 5. collection.json

```json
{
  "$schema": "../node_modules/@angular-devkit/schematics/collection-schema.json",
  "schematics": {
    "my-component": {
      "description": "สร้าง Angular component พร้อม boilerplate",
      "factory": "./my-component/index#myComponent",
      "schema": "./my-component/schema.json",
      "aliases": ["mc"]
    },
    "feature-module": {
      "description": "สร้าง feature module พร้อม routing",
      "factory": "./feature-module/index#featureModule",
      "schema": "./feature-module/schema.json"
    }
  }
}
```

---

## 6. Feature Module Schematic

```typescript
// src/feature-module/index.ts
import { Rule, SchematicContext, Tree, chain, apply, url, applyTemplates, move, mergeWith } from '@angular-devkit/schematics';
import { strings, normalize } from '@angular-devkit/core';

export interface FeatureModuleSchema {
  name: string;
  path?: string;
  routing?: boolean;
  lazy?: boolean;
}

export function featureModule(options: FeatureModuleSchema): Rule {
  return (tree: Tree, context: SchematicContext) => {
    context.logger.info(`สร้าง Feature Module: ${options.name}`);
    
    const moduleName = strings.dasherize(options.name);
    const targetPath = normalize(`${options.path || 'src/app/features'}/${moduleName}`);

    const templateSource = apply(url('./files'), [
      applyTemplates({
        ...strings,
        ...options,
        moduleName: strings.classify(options.name)
      }),
      move(targetPath)
    ]);

    return chain([
      mergeWith(templateSource),
      updateAppModule(options)
    ])(tree, context);
  };
}

function updateAppModule(options: FeatureModuleSchema): Rule {
  return (tree: Tree, context: SchematicContext) => {
    const appModulePath = 'src/app/app.module.ts';
    
    if (!tree.exists(appModulePath)) {
      context.logger.warn('ไม่พบ app.module.ts');
      return tree;
    }

    if (options.lazy) {
      // เพิ่ม lazy loading route
      updateAppRouting(tree, options);
    }
    
    return tree;
  };
}

function updateAppRouting(tree: Tree, options: FeatureModuleSchema): void {
  const routingPath = 'src/app/app-routing.module.ts';
  if (!tree.exists(routingPath)) return;

  const content = tree.read(routingPath)?.toString('utf-8') || '';
  const moduleName = strings.dasherize(options.name);
  
  const lazyRoute = `  {
    path: '${moduleName}',
    loadChildren: () => import('./features/${moduleName}/${moduleName}.module').then(m => m.${strings.classify(options.name)}Module)
  },`;

  const updated = content.replace(
    /const routes: Routes = \[/,
    `const routes: Routes = [\n${lazyRoute}`
  );
  
  tree.overwrite(routingPath, updated);
}
```

---

## 7. Build และใช้งาน

```bash
# Build schematics
npm run build

# Link สำหรับ local development
npm link

# ใช้งานใน Angular project
cd my-angular-project
npm link my-schematics

# รัน schematic
ng generate my-schematics:my-component user-profile
ng generate my-schematics:my-component dashboard --hasService --hasStore

# ใช้ alias
ng generate my-schematics:mc product-card
```

---

## 8. Testing Schematics

```typescript
// src/my-component/index_spec.ts
import { SchematicTestRunner, UnitTestTree } from '@angular-devkit/schematics/testing';
import * as path from 'path';

const collectionPath = path.join(__dirname, '../collection.json');

describe('my-component', () => {
  const runner = new SchematicTestRunner('my-schematics', collectionPath);
  
  let appTree: UnitTestTree;

  beforeEach(async () => {
    appTree = await runner.runExternalSchematicAsync(
      '@schematics/angular', 
      'workspace',
      { name: 'test', version: '17' }
    ).toPromise() as UnitTestTree;
    
    appTree = await runner.runExternalSchematicAsync(
      '@schematics/angular',
      'application',
      { name: 'test-app', projectRoot: '' }
    ).toPromise() as UnitTestTree;
  });

  it('สร้างไฟล์ component', async () => {
    const tree = await runner.runSchematicAsync(
      'my-component',
      { name: 'hello-world' },
      appTree
    ).toPromise();

    expect(tree.files).toContain('/src/app/hello-world/hello-world.component.ts');
    expect(tree.files).toContain('/src/app/hello-world/hello-world.component.html');
    expect(tree.files).toContain('/src/app/hello-world/hello-world.component.scss');
  });

  it('component มี class name ถูกต้อง', async () => {
    const tree = await runner.runSchematicAsync(
      'my-component',
      { name: 'user-profile' },
      appTree
    ).toPromise();

    const content = tree.readContent('/src/app/user-profile/user-profile.component.ts');
    expect(content).toContain('class UserProfileComponent');
    expect(content).toContain("selector: 'app-user-profile'");
  });
});
```

---

## สรุป

| ขั้นตอน | รายละเอียด |
|---------|-----------|
| Schema | กำหนด input options |
| Factory | สร้าง Rule functions |
| Templates | ไฟล์ที่จะ generate |
| Collection | ลงทะเบียน schematics |
| Build + Publish | `npm run build && npm publish` |

### Best Practices

1. ใช้ `strings` utility (dasherize, classify, camelize)
2. เขียน unit tests สำหรับ schematics
3. Validate inputs ใน schema.json
4. Log ข้อความที่เป็นประโยชน์
5. Handle edge cases (ไฟล์มีอยู่แล้ว, module ไม่พบ)
