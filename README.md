
## 项目初始结构分析

通常项目会分成三个运行环境：开发人员在本地跑的开发环境(dev)、测试人员用来做黑盒测试的测试环境(test)和线上运行的生产环境(production)。
简单起见，本文只考虑开发环境(dev)和生产环境(prod)，测试环境可以自行类比。
综上，webpack的配置需要有两套，同时两套配置必然会存在相同的部分，故新建目录与文件如下图：

![](http://img.blog.csdn.net/20170816213431219?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQva2luZ292/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

同时引入一个库webpack-merge用于合并base config和特定环境的config
``` 
cnpm i -D webpack-merge
```
## 开始写webpack config
webpack-config/base.js
```
module.exports = {
  //common config
};
```
webpack-config/dev.js, webpack-config/prod.js
```
const webpackMerge = require('webpack-merge');
const base = require('./base');
module.exports = webpackMerge(base, {
  //specific config
});
```
webpack.config.js
```
const devModule = require('./webpack-config/dev');
const prodModule = require('./webpack-config/prod');
let finalModule = {};
let ENV = process.env.NODE_ENV;     //此处变量可由命令行传入
switch (ENV) {
  case 'dev':
    finalModule = devModule;
    break;
  case 'prod':
    finalModule = prodModule;
    break;
  default:
    break;
}
module.exports = finalModule;
```
## 编写npm scripts，区分环境

package.json
```
{
  "name": "webpack-demo",
  "version": "1.0.0",
  "scripts": {
    "dev": "cross-env NODE_ENV=dev webpack",
    "prod": "cross-env NODE_env=prod webpack"
  },
  "devDependencies": {
    "webpack": "^2.2.0",
    "webpack-merge": "^2.6.1"
  }
}
```
由于*unix和windows设置NODE_ENV的语句有所差异，此处用到了一个库cross-env以达到兼容的目的
```
cnpm i -D cross-env
```
## 从dev环境写起
新建一个src目录用户存放项目源文件，同时在src下新建一个index.js作为打包的入口
webpack-dev-server
安装webpack-dev-server
```
    cnpm i -D webpack-dev-server
```
配置webpack config
webpack-config/dev.js
```
    ...
    module.exports = webpackMerge(base, {
      entry: process.cwd() + '/src/index.js',
      output: {
        filename: '[name].bundle.js'
      },
      devtool: 'eval-source-map'   //enable srouce map
    });
```
修改npm scripts
package.json
```
    {
     ...
     "scripts": {
       "dev": "cross-env NODE_ENV=dev webpack-dev-server --inline --hot --host 0.0.0.0",
       ...
     }
    }
```
## htmlWebpackPlugins
src目录下新建一个index.html作为template，htmlWebpackPlugins会根据这个template生成网站的index.html，同时自动写入bundle依赖。
src下放一个favicon.ico作为网站的icon
安装htmlWebpackPlugins
```
    cnpm i -D html-webpack-plugin
```
 配置webpack config
    webpack-config/dev.js
```
    ...
    const HtmlWebpackPlugin = require('html-webpack-plugin');
    ...
    module.exports = webpackMerge(base, {
      ...
      plugins: [
        new HtmlWebpackPlugin({
          filename: 'index.html',
          template: process.cwd() + '/src/index.html',
          favicon: process.cwd() + '/src/index.html'
        })
      ]
      ...
    });
```
到这一步算是完成了最基本的开发环境配置，命令行执行npm run dev，然后浏览器打开localhost:8080就能看到成果
将npm scripts中的start命令指向npm run dev，这样每次开始开发只需要执行npm start
package.json
```
{
  ...
  "scripts": {
    ...
    "start": "npm run dev"
    ...
  }
  ...
}
```
## Loaders(Rules)
webpack本身只能处理js模块，如果需要处理其他类型的文件，就需要Loaders进行转换
ES6+支持
ES6+虽然不能直接被浏览器全部识别，但是能用babel转换成ES5代码。
    安装babel编译相关依赖
```
    cnpm i -D babel-core babel-preset-latest babel-preset-stage-2 babel-runtime babel-plugin-transform-runtime babel-loader
```
   新建.babelrc文件并写入:
    .babelrc
```
    {
      "presets": ["latest", "stage-2"],
      "plugins": ["transform-runtime"]
    }
```
   配置rules
    webpack-config/dev.js
```
    module.exports = webpackMerge(base, {
      ...
      module: {
        rules: [
          ...
          {
            test: /\.js$/,
            exclude: [/node_modules/],
            loader: 'babel-loader'
          }
          ...
        ]
      }
      ...
    });
```
## css处理
现如今css预处理器已经成为前端开发的标配，Sass(Scss),Less和Stylus各行其道，个人偏好scss。
PostCss可以作为一个后处理器，实现为css自动添加浏览器前缀等功能

- 安装相关依赖
```
    cnpm i -D style-loader css-loader postcss-loader autoprefixer node-sass sass-loader
```
- 新建postcss.config.js
    postcss.config.js
```
    module.exports = {
      plugins: [
        require('autoprefixer')({browsers: ['last 2 versions', 'iOS 7', 'Firefox > 20']})
      ]
    };
```
- 配置rules
    webpack-config/dev.js
```
    module.exports = webpackMerge(base, {
      ...
      module: {
        rules: [
          ...
          {
            test: /\.scss$/,
            exclude: [/node_modules/],
            use: [
              'style-loader',
              {
                loader: 'css-loader',
                options: {
                  sourceMap: true
                }
              },
              'postcss-loader',
              {
                loader: 'sass-loader',
                options: {
                  sourceMap: true
                }
              }
            ]
          }
          ...
        ]
      }
      ...
    });
```
## html模板、图片等的处理
- 安装相关依赖
```
    cnpm i -D html-loader file-loader url-loader
```
- 配置rules
    webpack-config/dev.js
```
    module.exports = webpackMerge(base, {
      ...
      module: {
        rules: [
          ...
          {
            test: /\.html$/,
            loader: 'html-loader',
            options: {
              minimize: true
            }
          },
          {
            test: /\.(jpe?g|png|gif|svg)$/i,
            loader: 'url-loader',
            options: {
              limit: 10000,
              hash: 'sha512',
              publicPath: '/',
              name: 'assets/images/[hash].[ext]'
            }
          }
          ...
        ]
      }
      ...
    });
```
至此，dev环境配置完成
## 配置production环境
### 抽出公共部分

- entry可以共用，prod的output需要加上文件chunkhash用来刷新缓存,并将文件输出至dist目录
    webpack-config/dev.js
```
    - entry: process.cwd() + '/src/index.js',
```
   webpack-config/base.js
```
    module.exports = {
      entry: process.cwd() + '/src/index.js',
    };
```
  webpack-config/prod.js
```
    const webpackMerge = require('webpack-merge');
    const base = require('./base');
    module.exports = webpackMerge(base, {
      output: {
        filename: 'bundle.[chunkhash].js',
        path: process.cwd() + '/dist'
      },
    });
```
- HtmlWebpackPlugin公共
    webpack-config/dev.js
```
    -new HtmlWebpackPlugin({
    -  filename: 'index.html',
    -  template: process.cwd() + '/src/index.html',
    -  favicon: process.cwd() + '/src/index.html'
    -})
```
  webpack-config/base.js
```
    const HtmlWebpackPlugin = require('html-webpack-plugin');
    module.exports = {
      ...
      plugins: [
        new HtmlWebpackPlugin({
          filename: 'index.html',
          template: process.cwd() + '/src/index.html',
          favicon: process.cwd() + '/src/index.html'
        })
      ],
    };
```
- js、html和图片loader公共，prod的css loader需要使用ExtractTextPlugin将css从js中分离出来
    webpack-config/dev.js
```
    -{
    -  test: /\.js$/,
    -  exclude: [/node_modules/],
    -  loader: 'babel-loader'
    -}
    ...
    -{
    -  test: /\.html$/,
    -  loader: 'html-loader',
    -  options: {
    -    minimize: true
    -  }
    -},
    -{
    -  test: /\.(jpe?g|png|gif|svg)$/i,
    -  loader: 'url-loader',
    -  options: {
    -    limit: 10000,
    -    hash: 'sha512',
    -    publicPath: '/',
    -    name: 'assets/images/[hash].[ext]'
    -  }
    -}
```
webpack-config/base.js
```
    ...
    module.exports = {
      ...
      module: {
        rules: [
          {
            test: /\.js$/,
            exclude: [/node_modules/],
            loader: 'babel-loader'
          },
          {
            test: /\.html$/,
            loader: 'html-loader',
            options: {
              minimize: true
            }
          },
          {
            test: /\.(jpe?g|png|gif|svg)$/i,
            loader: 'url-loader',
            options: {
              limit: 10000,
              hash: 'sha512',
              publicPath: '/',
              name: 'assets/images/[hash].[ext]'
            }
          }
        ]
      }
    };
```
```
cnpm i -D extract-text-webpack-plugin@beta
```
webpack-config/prod.js
```
    ...
    const ExtractTextPlugin = require("extract-text-webpack-plugin");
    ...
    module: {
      rules: [
        {
          test: /\.scss$/,
          exclude: [/node_modules/],
          use: ExtractTextPlugin.extract({
            fallback: 'style-loader',
            use: [
              {
                loader: 'css-loader',
                options: {
                  minimize: true
                }
              },
              'postcss-loader',
              'sass-loader'
            ]
          })
        }
      ]
    },
    ...
    plugins: [
      new ExtractTextPlugin({
        filename: "bundle.[chunkhash].css"
      })
    ],
```
## production环境的其它处理
- 使用UglifyJsPlugin压缩js
```
cnpm i - D uglifyjs-webpack-plugin
```
webpack-config/prod.js
```
...
const UglifyJSPlugin = require('uglifyjs-webpack-plugin');
...
module.exports = webpackMerge(base, {
  ...
  const webpack = require('webpack');
  ...
  plugins: [
    ...
    new UglifyJSPlugin({
      compress: {
        warnings: false,
      },
      output: {
        comments: false
      }
    })
    ...
  ]
  ...
})
...
```
## 使用CleanWebpackPlugin清空output目录
构建前先清空，防止出现垃圾文件
```
cnpm i -D clean-webpack-plugin
```
webpack-config/prod.js
```
...
module.exports = webpackMerge(base, {
  ...
  plugins: [
    ...
    const CleanWebpackPlugin = require('clean-webpack-plugin');
    ...
    new CleanWebpackPlugin(['dist'], {
      root: process.cwd(),
      exclude: []
    })
    ...
  ]
  ...
})
...
```
至此，production环境配置完毕，同时抽出了公共部分
## 配置resolve.alias
开发的时候如果有一个很深的目录比如：src/a/b/c/d/, 然后在d目录下的一个模块需要引入a目录下的模块，需要这样写：import '../../../some-module'，为了方便可以配置一个为src目录配置一个alias，这样模块引入只需要这样写：import src/a/some-module。
  webpack-config/base.js
```
    ...
    const path = require('path');
    ...
    module.exports = {
      ...
      resolve: {
        extensions: ['.js'],
        alias: {
          src: path.resolve(__dirname, './../src')
        }
      }
      ...
    };
```
- 通常dev环境和production环境的配置参数比如api domain会有差异，所以需要利用alias将用户两个环境配置文件区分开来
    src目录下新建config目录，src/config目录下新增三个文件：base.js、dev.js、prod.js
    webpack-config/base.js
```
    export default {
      version: '1.0.0'
    }
```
  webpack-config/dev.js
```
    import base from './base'
    export default {
      ...base,
      env: 'dev'
    }
```
  webpack-config/prod.js
```
    import base from './base'
    export default {
      ...base,
      env: 'prod'
    }
```
- 配置alias
    webpack-config/dev.js

```
    ...
    const path = require('path');
    ...
    module.exports = webpackMerge(base, {
      ...
      resolve: {
        alias: {
          config: path.resolve(__dirname, './../src/config/dev.js')
        }
      }
      ...
    })
    ...
```
   webpack-config/prod.js
```
    ...
    const path = require('path');
    ...
    module.exports = webpackMerge(base, {
      ...
      resolve: {
        alias: {
          config: path.resolve(__dirname, './../src/config/prod.js')
        }
      }
      ...
    })
    ...
```
如果本文对你有用的话，不妨点击一下右上角的Star  ^_^

