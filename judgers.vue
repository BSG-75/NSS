<template>
    <div class="judgers-container">
        <div>
            <form v-if="!validValue.accessLevel" v-on:submit.prevent="login()">
                <input type="text" name="username" v-model="input.username" placeholder="鉴定员用户名">
                <input type="password" name="password" v-model="input.password" placeholder="密码">
                <input type="submit" value="登录">
            </form>
            <div v-else>
                当前身份：{{ validValue.username }}
                <button v-on:click="localToken = null; updateAccessLevel()">
                    退出
                </button>
            </div>
        </div>
        <table>
            <thead>
                <th>鉴定员名称</th>
                <th>个人信息</th>
                <th>
                    <div id="setJudger" v-if="validValue.accessLevel > 0">
                        <a href="admincontrol.html">编辑鉴定员</a>
                    </div>
                </th>
            </thead>
            <tbody>
                <tr v-for="judger in judgers" v-bind:key="judger.username">
                    <td>{{ judger.username }}</td>
                    <td colspan="2">{{ judger.description }}</td>
                </tr>
            </tbody>
        </table>
    </div>
</template>
<script>
module.exports = {
    data: function() {
        return {
            input: {
                username: '',
                password: '',
            },
            judgers: [],
            tokenValue: '',
        };
    },
    props: {
        value: {
            required: true,
            type: Object
        },
    },
    computed: {
        localToken: {
            get: function() {
                if(!this.tokenValue) {
                    const cookies = new Map(
                        decodeURIComponent(document.cookie)
                        .split(';') // 按照';'把 cookie 分成一个数组
                        .map(splitted => splitted.split('=', 2))
                        // 然后按照'='把每个部分再分成两部分（key 和 value）
                        // 并把这些 key 和 value 转换成一个 Map
                    );
                    this.tokenValue = (cookies.get('token') || '0');
                }
                return this.tokenValue;
            },
            set: function(value) {
                if(!value || value == '0') {
                    document.cookie = 'token=0; expires=Thu, 01 Jan 1970 00:00:01 GMT;';
                    this.tokenValue = null;
                    return;
                }

                let date = new Date();
                date.setDate(date.getDate() + 1);
                document.cookie = 'token=' + value + '; expires=' + date.toUTCString();
                this.tokenValue = value;
            }
        }, 
        validValue: function() {
            const valid = this.value && this.value.username && this.value.accessLevel;
            return valid ? this.value : { username: '游客', accessLevel: 0 };
        }
    },
    methods: {
        updateAccessLevel: function() {
            fetch('nss.php?do=getAccessLevel&token=' + this.localToken)
                .then(response => response.json())
                .then(response => {
                    // 设置数据
                    this.$emit('input', {
                        username: response.username,
                        accessLevel: response.accessLevel,
                        token: this.localToken
                    });
                });
        },
        login: function() {
            fetch('nss.php?do=login', {
                method: 'post',
                body: JSON.stringify(this.input),
                headers: {
                    'Content-type': 'application/json'
                }
            })
            .then(response => response.json())
            .then(result => {
                if(!result || !result.token || result.token == '0') {
                    throw new Error('登录失败');
                }
                this.localToken = result.token;
                this.updateAccessLevel();
            })
            .catch(reason => alert(reason));
        },
        listJudgers: function() {
            fetch('nss.php?do=getJudgers')
                .then(response => {
                    return response.json();
                })
                .then(response => {
                    // 设置数据
                    this.judgers = response.judgers;
                });
            setInterval(60000, this.listJudgers);
        },
    },
    mounted: function() {
        this.updateAccessLevel();
        this.listJudgers();
    }
}
</script>