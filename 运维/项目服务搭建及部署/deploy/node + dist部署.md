## 示例

```shell
# 切目录、拉代码、下依赖、打包、覆盖原文件
cd /home/developer/jdy-node/JDY-Admin-Frontend/
git pull
yarn
npm run build
rm -rf /home/developer/nginx/data/mechanism-admin
mv /home/developer/jdy-node/JDY-Admin-Frontend/dist/ /home/developer/nginx/data/mechanism-admin
```

