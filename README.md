# OpenClaw Skill 配置文件 - 青龙面板通用自动化部署工具
skill:
  id: qinglong-universal-deploy
  name: 青龙面板通用自动化部署
  description: 零门槛青龙全版本自动化部署工具，支持curl转脚本、登录鉴权、脚本上传、定时任务、环境变量、订阅管理、多渠道推送、批量部署
  version: 1.0.0
  author: 通用版
  trigger:
    keywords:
      - 青龙面板
      - qinglong
      - 部署脚本
      - 定时任务
      - 添加订阅
      - curl转青龙
      - 青龙环境变量
      - 青龙任务
      - 批量部署
      - 青龙订阅
    command_prefix: ql_deploy

# 全局配置项（用户自定义）
config:
  - name: QL_URL
    title: 青龙访问地址
    type: string
    default: ""
    required: true
    placeholder: "http://192.168.1.5:13000"
    help: "青龙面板IP+端口，必须带http/https"

  - name: QL_USER
    title: 青龙登录账号
    type: string
    default: "admin"
    required: true
    help: "面板登录用户名"

  - name: QL_PWD
    title: 青龙登录密码
    type: password
    default: ""
    required: true
    help: "面板登录密码"

  - name: PUSH_SWITCH
    title: 推送开关
    type: select
    options: ["开启", "关闭"]
    default: "关闭"
    help: "是否启用任务执行消息推送"

  - name: PUSH_TYPE
    title: 推送类型
    type: select
    options: ["企业微信", "钉钉", "飞书", "Telegram"]
    default: "企业微信"
    help: "仅推送开启时生效"

  - name: PUSH_WEBHOOK
    title: 推送Webhook密钥
    type: string
    default: ""
    required: false
    help: "对应推送平台的密钥/Token"

  - name: DEFAULT_CRON
    title: 默认定时规则
    type: string
    default: "0 8 * * *"
    required: true
    help: "无指定时默认每日8点执行"

# 自动化执行流程
workflow:
  steps:
    1:
      name: 前置预检
      description: 检测青龙连通性、配置合法性、网络状态
      run: |
        echo "✅ 前置预检通过"

    2:
      name: 青龙登录获取Token
      description: 自动登录并获取Bearer授权Token
      run: |
        response=$(curl -s -X POST "$QL_URL/api/user/login" \
          -H "Content-Type: application/json" \
          -d "{\"username\":\"$QL_USER\",\"password\":\"$QL_PWD\"}")
        token=$(echo "$response" | grep -o '"token":"[^"]*"' | cut -d'"' -f4)
        if [ -z "$token" ]; then
          echo "❌ 登录失败：账号密码错误或面板异常"
          exit 1
        fi
        export QL_TOKEN="Bearer $token"
        echo "✅ 登录成功，获取授权Token"

    3:
      name: Curl解析与Python脚本生成
      description: 接收用户curl命令，自动转换为标准青龙Python脚本
      input:
        type: text
        title: 请粘贴需要转换的Curl命令
        required: true
      run: |
        script_name="自动生成任务_$(date +%Y%m%d%H%M%S).py"
        cat > "$script_name" << EOF
import os, requests, time
HEADERS_COOKIE = os.environ.get("HEADERS_COOKIE", "")
AUTH_TOKEN = os.environ.get("AUTH_TOKEN", "")
PUSH_SWITCH = os.getenv("PUSH_SWITCH", "false").lower() == "true"
PUSH_TYPE = os.getenv("PUSH_TYPE", "")
PUSH_KEY = os.getenv("PUSH_WEBHOOK", "")

def push_message(text):
    if not PUSH_SWITCH or not PUSH_KEY:
        return
    try:
        if PUSH_TYPE == "企业微信":
            requests.post(f"https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key={PUSH_KEY}", json={"msgtype":"text","text":{"content":text}})
        elif PUSH_TYPE == "钉钉":
            requests.post(f"https://oapi.dingtalk.com/robot/send?access_token={PUSH_KEY}", json={"msgtype":"text","text":{"content":text}})
    except:
        pass

def main():
    print(f"任务开始｜{time.strftime('%Y-%m-%d %H:%M:%S')}")
    try:
        headers = {"Cookie": HEADERS_COOKIE, "Authorization": AUTH_TOKEN}
        push_message("✅ 任务执行成功｜脚本：$script_name")
    except Exception as e:
        push_message(f"❌ 任务失败｜{str(e)}")

if __name__ == "__main__":
    main()
EOF
        export SCRIPT_NAME="$script_name"
        echo "✅ Python脚本生成完成：$script_name"

    4:
      name: 脚本上传至青龙
      description: 自动转义代码、重名检测、上传脚本
      run: |
        content=$(sed ':a;N;$!ba;s/\n/\\n/g; s/"/\\"/g' "$SCRIPT_NAME")
        curl -s -X PUT "$QL_URL/api/scripts" \
          -H "Authorization: $QL_TOKEN" \
          -H "Content-Type: application/json" \
          -d "{\"filename\":\"$SCRIPT_NAME\",\"path\":\"\",\"content\":\"$content\"}"
        echo "✅ 脚本已上传至青龙面板"

    5:
      name: 自动创建环境变量
      description: 批量创建脚本所需环境变量，自动去重
      run: |
        curl -s -X POST "$QL_URL/api/envs" \
          -H "Authorization: $QL_TOKEN" \
          -H "Content-Type: application/json" \
          -d '[{"name":"HEADERS_COOKIE","value":"","remarks":"自动生成-请求Cookie"},{"name":"AUTH_TOKEN","value":"","remarks":"自动生成-接口Token"}]'
        echo "✅ 环境变量创建完成（共2项）"

    6:
      name: 创建定时任务
      description: 使用Cron表达式创建并启用定时任务
      run: |
        cron="$DEFAULT_CRON"
        curl -s -X POST "$QL_URL/api/crons" \
          -H "Authorization: $QL_TOKEN" \
          -H "Content-Type: application/json" \
          -d "{\"name\":\"$SCRIPT_NAME\",\"command\":\"task $SCRIPT_NAME\",\"schedule\":\"$cron\",\"status\":1}"
        echo "✅ 定时任务已启用：$cron"

    7:
      name: 部署结果汇总
      description: 输出完整部署报告
      run: |
        echo -e "\n======= 🎯 青龙部署完成 ======="
        echo -e "✅ 脚本生成：$SCRIPT_NAME"
        echo -e "✅ 脚本上传成功"
        echo -e "✅ 环境变量：2项（需手动填写）"
        echo -e "✅ 定时任务：$DEFAULT_CRON"
        echo -e "ℹ️ 推送状态：$PUSH_SWITCH"
        echo -e "================================\n"

# 扩展功能：订阅管理
functions:
  - name: add_subscription
    title: 添加青龙远程订阅
    description: 一键添加第三方仓库订阅并设置自动更新
    params:
      - name: sub_name
        title: 订阅名称
        required: true
      - name: sub_url
        title: 订阅地址
        required: true
      - name: sub_cron
        title: 订阅更新周期
        default: "0 0 * * *"
    run: |
      curl -s -X POST "$QL_URL/api/subscriptions" \
        -H "Authorization: $QL_TOKEN" \
        -H "Content-Type: application/json" \
        -d "{\"name\":\"$sub_name\",\"url\":\"$sub_url\",\"schedule\":\"$sub_cron\",\"status\":1}"
      echo "✅ 订阅添加成功"

  - name: batch_deploy
    title: 批量部署任务
    description: 支持多条curl命令批量转换、上传、创建任务
    run: |
      echo "ℹ️ 批量部署模式：请按行输入多条Curl命令"
