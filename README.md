# NgSchematics

> <https://angular.cn/guide/schematics-for-libraries>

## 创建原理图

```sh
npm install -g @angular-devkit/schematics-cli
schematics blank --name=hello-world
cd hello-world
npm install
npm run build
code .
```

### 在集合中添加原理图

```sh
schematics blank --name=goodbye-world
```

### 原理图包含文件

* index.ts 定义命名原理图中转换逻辑的代码。
* schema.json  原理图变量定义。
* schema.d.ts 原理图变量。
* files/ 要复制的可选组件/模板文件。

## 创建 lib 原理图合集

### 1. 创建 lib

```sh
## 全局安装 schematics 命令
npm install -g @angular-devkit/schematics-cli

## 创建 lib project
ng new my-workspace --create-application=false
cd my-workspace

## ng generate 命令会在你的工作空间中创建 projects/my-lib 文件夹
ng generate library my-lib

## 初始化安装
npm i
npm i -D  @angular-devkit/schematics @angular-devkit/core
```

### 2. 在 my-lib 下创建 schematics 原理图集

```sh
cd projects/my-lib
mkdir schematics
```

### 3. 创建原理图集配置文件 collection.json

```sh
cd schematics
touch collection.json
```

```json
// collection.json
{
    "$schema": "../../../node_modules/@angular-devkit/schematics/collection-schema.json",
    "schematics": {
        // 自定义原理图配置...
    }
}
```

#### schematics 参数说明

* aliases => 别名.
* factory => 定义代码.
* description => 简单的说明.
* schema => 你的 schema 的设置. 可以理解为用户输入的参数 schema 交互提示

### 4. 在这个库项目的 package.json 文件中，添加一个 'schematics' 的条目，里面带有你的模式定义文件的路径。当 Angular CLI 运行命令时，会根据这个条目在你的集合中查找指定名字的原理图

```json
// my-lib/package.json
{
  "schematics": "./schematics/collection.json",
}
```

### 5. 创建自定义原理图

```sh
## 默认被创建到 src 目录下, 可创建后手动改下文件位置到 schematics 目录下
schematics blank --name=my-service
## 创建后 collection.json 配置自动修改, 并创建 index.ts 
```

### 6. 创建 my-service schema.json

```json
{
    "$schema": "http://json-schema.org/schema",
    "id": "SchematicsMyService",
    "title": "My Service Schema",
    "type": "object",
    "properties": {
        "name": {
            "description": "The name of the service.",
            "type": "string"
        },
        "path": {
            "type": "string",
            "format": "path",
            "description": "The path to create the service.",
            "visible": false
        },
        "project": {
            "type": "string",
            "description": "The name of the project.",
            "$default": {
                "$source": "projectName"
            }
        }
    },
    "required": [
        "name"
    ]
}
```

#### 6.1 修改 collection.json 配置

```json
{
    "$schema": "../../node_modules/@angular-devkit/schematics/collection-schema.json",
    "schematics": {
        "my-service": {
            "description": "Generate a service in the project.",
            "factory": "./my-service/index#myService",
            "schema": "./my-service/schema.json"
        }
  }
}
```

配置说明见 步骤 3

### 7. my-lib 目录下创建 schematics ts config

```json
// tsconfig.schematics.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "lib": [
      "es2018",
      "dom"
    ],
    "declaration": true,
    "module": "commonjs",
    "moduleResolution": "node",
    "noEmitOnError": true,
    "noFallthroughCasesInSwitch": true,
    "noImplicitAny": true,
    "noImplicitThis": true,
    "noUnusedParameters": true,
    "noUnusedLocals": true,
    "rootDir": "schematics",
    "outDir": "../../dist/my-lib/schematics",
    "skipDefaultLibCheck": true,
    "skipLibCheck": true,
    "sourceMap": true,
    "strictNullChecks": true,
    "target": "es6",
    "types": [
      "jasmine",
      "node"
    ]
  },
  "include": [
    "schematics/**/*"
  ],
  "exclude": [
    "schematics/*/files/**/*"
  ]
}
```

### 8. 创建模板文件 files/*.template

```ts
// files/__name@dasherize__.service.ts.template
// 说明: __name@dasherize__ 这个文件名 依赖 `'@angular-devkit/core' 包下 strings.dasherize` 解析

import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({
  providedIn: 'root'
})
export class <%= classify(name) %>Service { 
  // 说明: class <%= classify(name) %> 依赖 `'@angular-devkit/core' 包下 strings.classify` 解析
  constructor(private http: HttpClient) { }
} 
```

### 9. coding index.ts 工厂函数

根据用户输入命令配置项(scheme.json), merge 模板文件和配置项创建生成文件

```ts
// index.ts
import {
  Rule, Tree, SchematicsException,
  apply, url, applyTemplates, move,
  chain, mergeWith
} from '@angular-devkit/schematics';
import { strings, normalize } from '@angular-devkit/core';
import { Schema as MyServiceSchema } from './schema';

export function myService(options: MyServiceSchema): Rule {
  return (tree: Tree) => {
    const workspaceConfig = tree.read('/angular.json');
    if (!workspaceConfig) {
      throw new SchematicsException('Could not find Angular workspace configuration');
    }

    // convert workspace to string
    const workspaceContent = workspaceConfig.toString();

    // parse workspace string into JSON object
    const workspace = JSON.parse(workspaceContent);

    if (!options.project) {
      options.project = workspace.defaultProject;
    }

    const projectName = options.project as string;

    const project = workspace.projects[projectName];

    const projectType = project.projectType === 'application' ? 'app' : 'lib';

    if (options.path === undefined) {
      options.path = `${project.sourceRoot}/${projectType}`;
    }

    const templateSource = apply(url('./files'), [
      applyTemplates({
        classify: strings.classify,
        dasherize: strings.dasherize,
        name: options.name
      }),
      move(normalize(options.path as string))
    ]);

    return chain([
      mergeWith(templateSource)
    ]);
  };
}
```

### 10. schematics build 自动化配置

```json
// my-lib/package.json
{
  "scripts": {
    "build": "../../node_modules/.bin/tsc -p tsconfig.schematics.json",
    "copy:schemas": "cp -P ./schematics/my-service/schema.json ../../dist/my-lib/schematics/my-service/schema.json",
    "copy:files": "cp -r schematics/my-service/files ../../dist/my-lib/schematics/my-service/files",
    "copy:collection": "cp -P schematics/collection.json ../../dist/my-lib/schematics/collection.json",
    "postbuild": "npm run copy:schemas && npm run copy:files && npm run copy:collection"
  },
  "peerDependencies": {
    "@angular/common": "^11.2.6",
    "@angular/core": "^11.2.6",
    "tslib": "^2.0.0"
  }
}
```

### 11. 构建产出 `my-lib` 库, schematics 原理图集

```sh
## cd 到项目根目录构建 my-lib 库
ng build my-lib

## cd 到 my-lib 产出 schematics 原理图集
cd projects/my-lib
npm run build
```

### 12. 链接这个库

```sh
npm link dist/my-lib
```

### 13. 运行原理图

```sh
ng generate my-lib:my-service --name my-data

## 控制台输出: CREATE src/app/my-data.service.ts
```

## 提供安装支持

作用: 增强用户的初始安装过程, 使用 SchematicContext 来触发安装任务。该任务会借助用户首选的包管理器将该库添加到宿主项目的 package.json 配置文件中，并将其安装到该项目的 node_modules 目录下

### 1. 创建 ng-add 原理图

```sh
cd projects/my-lib
schematics blank --name=ng-add
```

### 2. 修改 package.json 配置

```json
// my-lib/package.json
{
    "ng-add": {
        "save": "devDependencies"
    }
}
```

false - 不把此包添加到 package.json

true - 把此包添加到 dependencies

"dependencies" - 把此包添加到 dependencies

"devDependencies" - 把此包添加到 devDependencies

### 3. 开发工厂函数

```ts
// schematics/ng-add/index.ts
import { Rule, SchematicContext, Tree } from '@angular-devkit/schematics';
import { NodePackageInstallTask } from '@angular-devkit/schematics/tasks';

// Just return the tree
export function ngAdd(options: any): Rule {
    return (tree: Tree, context: SchematicContext) => {
        context.addTask(new NodePackageInstallTask());
        return tree;
    };
}
```

### 4. build

```sh
ng build my-lib
cd projects/my-lib
npm run build
npm link dist/my-lib
```

### 5. 安装

```sh
## FIXME: ?? 没效果
ng generate my-lib:ng-add 
```
