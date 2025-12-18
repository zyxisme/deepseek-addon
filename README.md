# deepseek-addon
dicksuck生成的神秘插件


```php
<?php
// /modules/servers/xboard/xboard.php

use think\Db;

define('XBOARD_DEBUG', true);

function xboard_debug($message, $data = null)
{
    if (!XBOARD_DEBUG) return;
    $log = '[XBOARD-DEBUG] ' . $message;
    if ($data !== null) {
        $log .= ' | Data: ' . json_encode($data, JSON_UNESCAPED_UNICODE);
    }
    error_log($log);
}

function xboard_MetaData()
{
    return [
        'DisplayName' => 'XBoard/V2Board对接插件',
        'APIVersion'  => '1.0',
        'HelpDoc'     => 'https://github.com/v2board/v2board',
    ];
}

function xboard_ConfigOptions()
{
    return [
        [
            'type'        => 'text',
            'name'        => '商品ID',
            'placeholder' => '请输入XBoard中的商品ID',
            'description' => '在XBoard后台创建的商品ID',
            'default'     => '',
            'key'         => 'plan_id'
        ],
        [
            'type'        => 'text',
            'name'        => '月流量限制(GB)',
            'placeholder' => '请输入流量限制',
            'description' => '每月可用流量，单位GB',
            'default'     => '100',
            'key'         => 'monthly_traffic_limit'
        ],
        [
            'type'        => 'text',
            'name'        => '设备数限制',
            'placeholder' => '请输入设备数限制',
            'description' => '同时在线设备数限制',
            'default'     => '3',
            'key'         => 'device_limit'
        ],
        [
            'type'        => 'text',
            'name'        => '速度限制(Mbps)',
            'placeholder' => '请输入速度限制',
            'description' => '最大连接速度限制',
            'default'     => '100',
            'key'         => 'speed_limit'
        ],
        [
            'type'        => 'dropdown',
            'name'        => '协议类型',
            'description' => '支持的代理协议',
            'options'     => [
                'vmess'    => 'VMess',
                'vless'    => 'VLESS',
                'trojan'   => 'Trojan',
                'shadowsocks' => 'Shadowsocks',
                'hysteria' => 'Hysteria'
            ],
            'default'     => 'vmess',
            'key'         => 'protocol'
        ],
        [
            'type'        => 'yesno',
            'name'        => '自动开通',
            'description' => '是否自动在XBoard开通服务',
            'default'     => '1',
            'key'         => 'auto_create'
        ],
        [
            'type'        => 'textarea',
            'name'        => '节点列表',
            'placeholder' => '每行一个节点ID或名称',
            'description' => '分配给此产品的节点（每行一个）',
            'rows'        => 5,
            'default'     => '',
            'key'         => 'node_ids'
        ]
    ];
}

function xboard_TestLink($params)
{
    xboard_debug('测试XBoard API连接', $params);
    
    $apiResult = xboard_ApiRequest($params, '/api/v1/admin/user/list', [], 'GET', true);
    
    xboard_debug('测试连接响应', $apiResult);
    
    if ($apiResult === null) {
        return [
            'status' => 200,
            'data'   => [
                'server_status' => 0,
                'msg'           => '连接失败: 无法连接到XBoard API'
            ]
        ];
    }
    
    if (isset($apiResult['success']) && $apiResult['success'] === true) {
        return [
            'status' => 200,
            'data'   => [
                'server_status' => 1,
                'msg'           => '连接成功，XBoard版本: ' . ($apiResult['version'] ?? '未知')
            ]
        ];
    }
    
    $errorMsg = isset($apiResult['message']) ? $apiResult['message'] : '未知错误';
    return [
        'status' => 200,
        'data'   => [
            'server_status' => 0,
            'msg'           => '连接失败: ' . $errorMsg
        ]
    ];
}

function xboard_ApiRequest($params, $endpoint, $data = [], $method = 'POST', $isAdmin = false)
{
    $apiUrl = rtrim($params['server_host'], '/');
    if (empty($apiUrl)) {
        $apiUrl = 'http://' . $params['server_ip'];
        if (!empty($params['port'])) {
            $apiUrl .= ':' . $params['port'];
        }
    }
    
    $url = $apiUrl . $endpoint;
    
    // 根据是否是管理API使用不同的认证方式
    $headers = [
        'Content-Type: application/json',
        'Accept: application/json',
    ];
    
    if ($isAdmin) {
        // 管理员API使用API Key认证
        $headers[] = 'Authorization: Bearer ' . $params['accesshash'];
    } else {
        // 用户API使用用户token
        $headers[] = 'X-Token: ' . $params['server_password'];
    }
    
    xboard_debug('API请求详情', [
        'url' => $url,
        'method' => $method,
        'isAdmin' => $isAdmin,
        'headers' => $headers
    ]);
    
    $curl = curl_init();
    $curlOptions = [
        CURLOPT_URL            => $url,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_ENCODING       => '',
        CURLOPT_MAXREDIRS      => 10,
        CURLOPT_TIMEOUT        => 30,
        CURLOPT_CONNECTTIMEOUT => 10,
        CURLOPT_FOLLOWLOCATION => true,
        CURLOPT_HTTP_VERSION   => CURL_HTTP_VERSION_1_1,
        CURLOPT_CUSTOMREQUEST  => $method,
        CURLOPT_HTTPHEADER     => $headers,
    ];
    
    if ($params['secure']) {
        $curlOptions[CURLOPT_SSL_VERIFYPEER] = false;
        $curlOptions[CURLOPT_SSL_VERIFYHOST] = false;
    }
    
    if ($method === 'POST' || $method === 'PUT') {
        if (!empty($data)) {
            $curlOptions[CURLOPT_POSTFIELDS] = json_encode($data);
        }
    }
    
    curl_setopt_array($curl, $curlOptions);
    
    $response = curl_exec($curl);
    $errno = curl_errno($curl);
    $httpCode = curl_getinfo($curl, CURLINFO_HTTP_CODE);
    
    if ($errno) {
        xboard_debug('CURL错误', [
            'errno' => $errno,
            'error' => curl_error($curl)
        ]);
        curl_close($curl);
        return null;
    }
    
    curl_close($curl);
    
    xboard_debug('API原始响应', [
        'http_code' => $httpCode,
        'response' => $response
    ]);
    
    $decoded = json_decode($response, true);
    return $decoded;
}

function xboard_CreateAccount($params)
{
    xboard_debug('开始创建XBoard账户', [
        'domain' => $params['domain'],
        'email' => $params['user_info']['email'] ?? ''
    ]);
    
    // 1. 先检查用户是否已存在
    $userCheck = xboard_FindOrCreateUser($params);
    if (!$userCheck['success']) {
        return '创建用户失败: ' . $userCheck['message'];
    }
    
    $userId = $userCheck['user_id'];
    
    // 2. 为用户创建订单并开通服务
    $orderResult = xboard_CreateOrder($params, $userId);
    if (!$orderResult['success']) {
        return '创建订单失败: ' . $orderResult['message'];
    }
    
    // 3. 获取订阅链接
    $subscribeResult = xboard_GetSubscribeLink($params, $userId);
    if (!$subscribeResult['success']) {
        return '获取订阅链接失败: ' . $subscribeResult['message'];
    }
    
    // 保存订阅链接到主机备注
    xboard_SaveSubscribeInfo($params['hostid'], $subscribeResult['data']);
    
    return 'success';
}

function xboard_FindOrCreateUser($params)
{
    $userEmail = $params['user_info']['email'] ?? '';
    $userName = $params['user_info']['username'] ?? $params['username'] ?? 'user_' . $params['userid'];
    
    // 首先尝试查找用户
    $findResult = xboard_ApiRequest($params, '/api/v1/admin/user/fetch', [
        'email' => $userEmail
    ], 'POST', true);
    
    if ($findResult && isset($findResult['success']) && $findResult['success'] === true) {
        if (!empty($findResult['data'])) {
            xboard_debug('找到已存在的用户', $findResult['data']);
            return [
                'success' => true,
                'user_id' => $findResult['data']['id']
            ];
        }
    }
    
    // 用户不存在，创建新用户
    $createData = [
        'email' => $userEmail,
        'password' => $params['password'] ?? 'defaultPassword123',
        'plan_id' => $params['config_option1'] ?? 0, // 商品ID
        'balance' => 0,
        'commission_rate' => 0,
        'discount' => 1,
        'is_admin' => false,
        'is_staff' => false,
        'uuid' => xboard_GenerateUUID(),
        'remarks' => 'Created from Zhijian MF',
        'invite_user_id' => null
    ];
    
    $createResult = xboard_ApiRequest($params, '/api/v1/admin/user/save', $createData, 'POST', true);
    
    xboard_debug('创建用户响应', $createResult);
    
    if ($createResult && isset($createResult['success']) && $createResult['success'] === true) {
        return [
            'success' => true,
            'user_id' => $createResult['data']['id']
        ];
    }
    
    return [
        'success' => false,
        'message' => $createResult['message'] ?? '创建用户失败'
    ];
}

function xboard_CreateOrder($params, $userId)
{
    $planId = $params['config_option1'] ?? 0;
    $productId = $params['productid'];
    
    // 创建订单数据
    $orderData = [
        'user_id' => $userId,
        'plan_id' => $planId,
        'period' => xboard_ConvertBillingCycle($params['billingcycle']),
        'total_amount' => 0, // 价格由XBoard计算
        'status' => 1, // 已支付
        'paid_at' => date('Y-m-d H:i:s'),
        'commission_status' => 0,
        'callback_no' => 'MF-' . $productId . '-' . time()
    ];
    
    $orderResult = xboard_ApiRequest($params, '/api/v1/admin/order/save', $orderData, 'POST', true);
    
    xboard_debug('创建订单响应', $orderResult);
    
    if ($orderResult && isset($orderResult['success']) && $orderResult['success'] === true) {
        // 开通服务
        $serviceData = [
            'order_id' => $orderResult['data']['id'],
            'plan_id' => $planId,
            'user_id' => $userId,
            'period' => $orderData['period'],
            'expired_at' => date('Y-m-d H:i:s', strtotime('+' . $orderData['period'] . ' months')),
            'u' => 0,
            'd' => 0,
            'transfer_enable' => ($params['config_option2'] ?? 100) * 1024 * 1024 * 1024, // GB to bytes
            'device_limit' => $params['config_option3'] ?? 3,
            'speed_limit' => ($params['config_option4'] ?? 100) * 1024 * 1024, // Mbps to bytes
            'status' => 1 // 启用
        ];
        
        $serviceResult = xboard_ApiRequest($params, '/api/v1/admin/service/save', $serviceData, 'POST', true);
     
```