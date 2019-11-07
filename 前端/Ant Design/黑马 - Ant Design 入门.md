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

```javascript
// 参考官方示例，进行使用
import React from 'react'
import { Tabs } from 'antd'

const TabPane = Tabs.TabPane;

const callback = (key) => {
    console.log(key);
}

class MyTabs extends React.Component {
    
    render() {
		return (
        	<Tabs defaultActiveKey="1" onChange={ callback }>
            	<TabPane tab="Tab 1" key="1"> Content of Tab Pane1 </TabPane>
				<TabPane tab="Tab 2" key="2"> Content of Tab Pane2 </TabPane>	
				<TabPane tab="Tab 3" key="3"> Content of Tab Pane3 </TabPane>
			</Tabs>
        )
    }
    
} 

export default MyTabs;
```

## 布局

```shell
# antd 布局: https://ant.design/components/layout-cn/
```

### 组件概述

```shell
# Layout:
	# 布局容器，其下可嵌套 Header Sider Content Footer 或 Layout 本身，可以放在任何父容器中。
	
# Header:
	# 顶部布局，自带默认样式，其下可嵌套任何元素，只能放在 Layout 中。
	
# Sider:
	# 侧边栏，自带默认样式及基本功能，其下可嵌套任何元素，只能放在 Layout 中。
	
# Content:
	# 内容部分，自带默认样式，其下可嵌套任何元素，只能放在 Layout 中。
	
# Footer:
	# 底部布局，自带默认样式，其下可嵌套任何元素，只能放在 Layout 中。
```

### 搭建整体框架

```javascript
// 在 src 目录下创建 layouts 目录，并且在 layouts 目录下创建 index.js 文件
// 特别说明: 在 umi 中约定的目录结构中，layouts/index.js 文件将会被作为全局的布局文件
import React from 'react'
import { Layout } from 'antd';

const { Header, Footer, Sider, Content } = Layout;

class BasicLayout extends React.Component {
    
    render() {
        return (
        	<Layout>
            	<Sider> Sider </Sider>
            	<Layout>
            		<Header> Header </Header>
            		<Content> Content </Content>
            		<Footer> Footer </Footer>
            	</Layout>
            </Layout>
        );
    }
    
}

export default BasicLayout;
```

```javascript
// 接下来，配置路由:<非必须 config.js>
export default {
    plugins: [
        ['umi-plugin-react', {
            dva: true, // 默认开启 dva 功能
            antd: true // 开启 Ant Design 功能
        }]
    ],
    routes: [{
        path: '/',
        component: '../layouts' // 配置布局路由
    }]
}
```

### 子页面使用布局

```shell
# 前面所定义的布局是全局布局，在子页面中如何使用这个全局布局呢？

# 首先，需要在布局文件中，将 Content 内容替换成 {this.props.children}, 意思是引入传递的内容。
```

```javascript
import React from 'react'
import { Layout } from 'antd';

const { Header, Footer, Sider, Content } = Layout;

class BasicLayout extends React.Component {
    
    render() {
        return (
        	<Layout>
            	<Sider> Sider </Sider>
            	<Layout>
            		<Header> Header </Header>
            		<Content> {this.props.children} </Content>
            		<Footer> Footer </Footer>
            	</Layout>
            </Layout>
        );
    }
    
}

export default BasicLayout;
```

```shell
# 接下来配置路由 <在布局路由下面进行配置> :
	# 说明: 下面的路由配置，表明通过手动配置的方式上进行访问页面，而不采用 umi 默认的路由方式
```

```javascript
export default {
    plugins: [
        ['umi-plugin-react', {
            dva: true, // 开启 dva 功能
            antd: true // 开启 Ant Desgin 功能
        }]
    ],
    routes: [{
        path: '/',
        component: '../layouts', // 配置布局路由
        routes: [ // 在这里进行配置子页面
            {
                path: '/myTabs',
                component: './myTabs'
            }
        ]
    }]
};
```

### 美化页面

```javascript
// 接下来，对页面做一些美化的工作:
import React from 'react'
import { Layout } from 'antd';

const { Header, Footer, Sider, Content } = Layout;

class BasicLayout extends React.Component {
    
    render() {
        return (
        	<Layout>
            	<Sider wieth={256} style={{ minHeight: '100vh', color: 'white'}}>
            		Sider
				</Sider>
			<Layout>
            	<Header style={{ background: '#fff', textAlign: 'center', padding: 0 }}	
               		Header
                </Header>

				<Content style={{ margin: '24px 16px 0' }}>
                	<div style={{ padding: 24, background: '#fff', minHeight: 360 }}>
                    	{this.props.children}
                    </div>
				</Content>

				<Footer style={{ textAlign: 'center' }}>
                    Swordsman 后台系统 Created by AgeFades
                </Footer>
			</Layout>
        );
    }
    
}

export default BasicLayout;
```

### 引入导航条

```javascript
// 使用 Menu 组件作为导航条 https://ant.design/components/menu-cn/
import React from 'react'
import { Layout, Menu, Icon } from 'antd';

const { Header, Footer, Sider, Content } = Layout;
const SubMenu = Menu.SubMenu;

class BasicLayout extends React.Componet {
    
    constructor(props){
        super(props);
        this.state = {
            collapsed: false
        }
    }
    
    render() {
        return (
        	<Layout>
            	<Sider width={256} style={{ minHeight: '100vh', color: 'white' }}>
            		<div style={{ height: '32px', background:'rgba(255,255,255,.2)', margin: '16px' }} />
					<Menu
						defaultSelectedKeys={['2']}
						defaultOpenKeys={['sub1']}
						mode="inline"
						theme="dark"
						inlineCoolapsed={ this.state.collapsed }
					>
                    	<Menu.Item key="1">
                            <Icon type="pie-chart" />
                            <span> Option 1 </span>
						</Menu.Item>
						<Menu.Item key="2">
                            <Icon type="desktop" />
                            <span> Option 2 </span>
						</Menu.Item>
						<Menu.Item key="3">
                            <Icon type="inbox" />
                            <span> Option 3 </span>
						</Menu.Item>
                            <SubMenu key="sub1" title={
								<span> 
                                	<Icon type="mail"/> 
                                	<span> Navigation One </span>
                                </span>
                            }>
                                <Menu.Item key="5"> Option 5 </Menu.Item>
                                <Menu.Item key="6"> Option 6 </Menu.Item>
                                <Menu.Item key="7"> Option 7 </Menu.Item>
                                <Menu.Item key="8"> Option 8 </Menu.Item>
                            </SubMenu>
                            <SubMenu key="sub2" title={
                                <span>
                                	<Icon type="appstore"/>
                                	<span> Navigation Two </span>
                                </span>
                            }>
                                <Menu.Item key="9"> Option 9 </Menu.Item>
								<Menu.Item key="10"> Option 10 </Menu.Item>
								<SubMenu key="sub3" title="Submenu">
                                	<Menu.Item key="11"> Option 11 </Menu.Item>
									<Menu.Item key="12"> Option 12 </Menu.Item>
								</SubMenu>
							</SubMenu>
						</Menu>
					</Sider>
					<Layout>
                    	<Header style={{background: '#fff', textAlign: 'center', padding: 0}} Header </Header>
						<Content style={{margin: '24px 16px 0'}}>
                            <div style={{padding: 24, background: '#fff', minHeight: 360}}> {this.props.children} </div>
						</Content>
						<Footer style={{textAlign: 'center'}}> Swordsman 后台系统 Created by AgeFades </Footer>
				</Layout>
			</Layout>
        );
    }
    
}

export default BasicLayout;
```

