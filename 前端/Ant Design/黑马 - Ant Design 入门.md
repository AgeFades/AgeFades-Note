# 黑马 - Ant Design 入门

## 简介

```shell
# Ant Design 是阿里蚂蚁金服团队基于 React 开发的 ui 组件，主要用于中后台系统的使用。

# 官网: https://ant.design/index-cn

# 特性:
	# 提炼自企业级中后台产品的交互语言和视觉风格
	# 开箱即用的高质量 React 组件
	# 使用 TypeScript 构建，提供完成的类型定义文件
	# 全链路开发和设计工具体系
```

## 开始使用

### 引入 Ant Design

```shell
# 在 umi 中，可以通过插件集 umi-plugin-react 中配置 antd 打开 antd 插件，antd 插件会引入 antd 并实现按需加载
```

```javascript
// 在 config.js 文件中进行配置
export default {
    plugins: [
        ['umi-plugin-react', {
            dva: true, // 开启 dva 功能
            antd: true // 开启 Ant Design 功能
        }]
    ]
};
```

### 小试牛刀

```shell
# 开始使用 antd 的组件，以 tabs 组件为例，地址: https://ant.design/components/tabs-cn/
```

```javascript
// 官方使用示例
import { Tabs } from 'antd';

const TabPane = Tabs.TabPane;

function callback(key) {
    console.log(key);
}

ReactDOM.render(
	<Tabs defaultActiveKey="1" onChange={callback}>
    	<TabPane tab="Tab 1" key="1"> Content of Tab Pane 1 </TabPane>
		<TabPane tab="Tab 2" key="2"> Content of Tab Pane 2 </TabPane>
		<TabPane tab="Tab 3" key="3"> Content of Tab Pane 3 </TabPane>
    </Tabs>,
	mountNode
);
```

