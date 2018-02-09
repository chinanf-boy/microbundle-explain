# microbundle

由`Rollup`提供支持的小型模块的零配置打包程序。by [developit](https://github.com/developit)

[![explain](http://llever.com/explain.svg)](https://github.com/chinanf-boy/Source-Explain)
    
Explanation

> "version": "1.0.0"

[github source](https://github.com/developit/microbundle)

~~[english](./README.en.md)~~

---

懒惰才能使人进步😊

---

## 目录

---

## 使用

``` js
microbundle
// 
microbundle build
```

默认情况下, 会在项目根目录-package.json- `main` 字样, 当然你也可以指定

> 你可以看看作者对 此项目-microbundle 的[使用项目-greenlet](https://github.com/developit/greenlet/blob/master/package.json#L10)

> 更可以看看-[greenlet-explain](https://github.com/chinanf-boy/greenlet-explain)

---

## package

``` js
  "main": "dist/microbundle.js", // 构建的
  "source": "src/index.js", // 源码
  "bin": "dist/cli.js", // 终端命令
```

## cli

既然💪-microbundle-💪 是通过
`microbundle` 启动构建的

我们自然从命令代码开始

``` js
#!/usr/bin/env node

import sade from 'sade'; // 命令解析
import microbundle from '.';

let { version } = require('../package');
let prog = sade('microbundle');

let toArray = val => Array.isArray(val) ? val : val == null ? [] : [val];
// 变 数组

prog
	.version(version)
	.option('--entry, -i', 'Entry module(s)')
	.option('--output, -o', 'Directory to place build files into')
	.option('--format, -f', 'Only build specified formats', 'es,cjs,umd')
	.option('--target', 'Specify your target environment', 'node')
	.option('--external', `Specify external dependencies, or 'all'`)
	.option('--compress', 'Compress output using UglifyJS', true)
	.option('--strict', 'Enforce undefined global context and add "use strict"')
	.option('--name', 'Specify name exposed in UMD builds')
	.option('--cwd', 'Use an alternative working directory', '.');

// 第一个变量 命令选项 第二个变量 说明 第三个变量 默认值

prog
	.command('build [...entries]', '', { default: true })
	.describe('Build once and exit')
	.action(run);

prog
	.command('watch [...entries]')
	.describe('Rebuilds on any change')
	.action((str, opts) => run(str, opts, true));

// Parse argv; add extra aliases
prog.parse(process.argv, {
	alias: {
		o: ['output', 'd'],
		i: ['entry', 'entries', 'e']
	}
});

// 这个命令解析库-api-其实都挺好懂的

// 主要是 run 

function run(str, opts, isWatch) {
	opts.watch = !!isWatch;
	opts.entries = toArray(str || opts.entry).concat(opts._);
	microbundle(opts) // 用 sade 解析来的命令给予 microbundle
		.then( output => {
            if (output!=null) process.stdout.write(output + '\n');
            // 把有色的结果-输出终端
			if (!opts.watch) process.exit(0);
		})
		.catch(err => {
            process.stderr.write(String(err) + '\n');
            // 把有色的结果-输出终端            
			process.exit(err.code || 1);
		});
}

```
## index

我们先直入主题 [`microbundle`](#microbundle-async) | [从头开始-代码1](#导入)

### 导入

代码 1- 22

``` js
import 'acorn-jsx';
import fs from 'fs';
import { resolve, relative, dirname, basename, extname } from 'path';
import chalk from 'chalk'; // 终端-有色-输出
import { map, series } from 'asyncro';
import promisify from 'es6-promisify';
import glob from 'glob';
import autoprefixer from 'autoprefixer';
import { rollup, watch } from 'rollup';
import nodent from 'rollup-plugin-nodent';
import commonjs from 'rollup-plugin-commonjs';
import nodeResolve from 'rollup-plugin-node-resolve';
import buble from 'rollup-plugin-buble';
import uglify from 'rollup-plugin-uglify';
import postcss from 'rollup-plugin-postcss';
import alias from 'rollup-plugin-strict-alias';
import gzipSize from 'gzip-size';
import prettyBytes from 'pretty-bytes';
import shebangPlugin from 'rollup-plugin-preserve-shebang';
import typescript from 'rollup-plugin-typescript';
import flow from './lib/flow-plugin';
import camelCase from 'camelcase';
```

### index-变量

``` js
const readFile = promisify(fs.readFile);
const stat = promisify(fs.stat);
const isDir = name => stat(name).then( stats => stats.isDirectory() ).catch( () => false );
const isFile = name => stat(name).then( stats => stats.isFile() ).catch( () => false );
const removeScope = name => name.replace(/^@.*\//, '');
const safeVariableName = name => camelCase(removeScope(name).toLowerCase().replace(/((^[^a-zA-Z]+)|[^\w.-])|([^a-zA-Z0-9]+$)/g, ''));

const WATCH_OPTS = {
	exclude: 'node_modules/**'
};
```

### microbundle-async

定义-异步-主函数

代码 35-135

``` js
export default async function microbundle(options) {
	let cwd = options.cwd = resolve(process.cwd(), options.cwd), // 命令路径
		hasPackageJson = true;

	try {
		options.pkg = JSON.parse(await readFile(resolve(cwd, 'package.json'), 'utf8')); // 路径中的 - package.json - 文件
	}
	catch (err) {
		process.stderr.write(chalk.yellow(`${chalk.yellow.inverse('WARN')} no package.json found. Assuming a name of "${basename(options.cwd)}".`)+'\n');
		let msg = String(err.message || err);
		if (!msg.match(/ENOENT/)) console.warn(`  ${chalk.red.dim(msg)}`);
		options.pkg = {};
		hasPackageJson = false;
	}

	if (!options.pkg.name) {
		options.pkg.name = basename(options.cwd); // 项目名
		if (hasPackageJson) {
			process.stderr.write(chalk.yellow(`${chalk.yellow.inverse('WARN')} missing package.json "name" field. Assuming "${options.pkg.name}".`)+'\n');
		}
	}

	const jsOrTs = async filename => resolve(cwd, `${filename}${await isFile(resolve(cwd, filename+'.ts')) ? '.ts' : '.js'}`);
    // 是 js 还是 ts 文件
    options.input = [];
    
	[].concat(
		options.entries && options.entries.length ? options.entries : options.pkg.source || (await isDir(resolve(cwd, 'src')) && await jsOrTs('src/index')) || await jsOrTs('index') || options.pkg.module
	).map( file => glob.sync(resolve(cwd, file)) ).forEach( file => options.input.push(...file) );
    
    //  从 定义的目录 or 文件 或 默认中找出-入口文件


    let main = resolve(cwd, options.output || options.pkg.main || 'dist');
    
    // 输出位置|文件-初始化

	if (!main.match(/\.[a-z]+$/) || await isDir(main)) {
        // 当 main不符合文件名规范 或者 是目录
        main = resolve(main, `${removeScope(options.pkg.name)}.js`);
        // 重定义-输出文件
    }

    
	options.output = main;

	let entries = (await map([].concat(options.input), async file => {
		file = resolve(cwd, file);
		if (await isDir(file)) {
			file = resolve(file, 'index.js');
		}
		return file; // 加入目录
	})).filter( (item, i, arr) => arr.indexOf(item)===i ); // 去掉多余

	options.entries = entries;

	options.multipleEntries = entries.length>1;

	let formats = (options.format || options.formats).split(','); // 是否有-构建输出文件类型-要求
    // always compile cjs first if it's there:
    // 总是构建 cjs 类型 优先
	formats.sort( (a, b) => a==='cjs' ? -1 : a>b ? 1 : 0);

	let steps = [];
	for (let i=0; i<entries.length; i++) {
		for (let j=0; j<formats.length; j++) {
            steps.push(createConfig(options, entries[i], formats[j], i===0 && j===0));
            // 根据每个入口文件, 每个输出类型
            // 带配置->一步一步添加
		}
	}

```

- [createConfig 配置好 rollup构建所需配置](./createConfig.readme.md)

---

``` js
	async function getSizeInfo(code, filename) {
		let size = await gzipSize(code);
		let prettySize = prettyBytes(size);
		let color = size < 5000 ? 'green' : size > 40000 ? 'red' : 'yellow';
        return `${' '.repeat(10-prettySize.length)}${chalk[color](prettySize)}: ${chalk.white(basename(filename))}`;
        // 显示-文件大小
	}

	if (options.watch) { // 监听
		const onBuild = options.onBuild;
		return new Promise( (resolve, reject) => {
			process.stdout.write(chalk.blue(`Watching source, compiling to ${relative(cwd, dirname(options.output))}:\n`));
			steps.map( options => {
				watch(Object.assign({
					output: options.outputOptions,
					watch: WATCH_OPTS
				}, options.inputOptions)).on('event', e => {
					if (e.code==='ERROR' || e.code==='FATAL') {
						return reject(e);
					}
					if (e.code==='END') {
						getSizeInfo(options._code, options.outputOptions.file).then( text => {
							process.stdout.write(`Wrote ${text.trim()}\n`);
						});
						if (typeof onBuild=='function') {
							onBuild(e);
						}
					}
				});
			});
		});
	}

```

---

``` js
	let cache;
	let out = await series(steps.map( ({ inputOptions, outputOptions }) => async () => {
		inputOptions.cache = cache;
		let bundle = await rollup(inputOptions); // 入口配置
		cache = bundle;
		await bundle.write(outputOptions); // 写 输出
        return await getSizeInfo(bundle._code, outputOptions.file);
        // 显示-文件大小, 并返回-颜色字符
	}));


    return chalk.blue(`Build output to ${relative(cwd, dirname(options.output)) || '.'}:`) + '\n   ' + out.join('\n   ');
    // 颜色记录
}
```