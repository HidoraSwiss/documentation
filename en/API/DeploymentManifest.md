# Deploy manifest from Jelastic's API

It can be useful to launch a deployment of an environment with a manifest from Jelatic's API.
However, the documentation of the API is not really clear and the the method which allow this is not documented.

The endpoint of the API that need to be used is <https://app.hidora.com/1.0/development/scripting/rest/eval>.

Find below an example of parameters to use:

```json
{
  "script": "MyScriptName",
  "session": "API session ID"
  "shortdomain": "my-env-name",
  "envName": "my-env-name",
  "type": "install",
  "name": "My Env",
  "displayName": "My Env",
  "region": "vz7",
  "settings": {...},
  "manifest": {...}
}
```

## Example in PHP

This PHP script deploys a server Xonotic on *my-xonotic.hidora.com* after each launch.

>Be careful, the variable $ENV_NAME needs to be different after each launch, otherwise you will have two environments with the same name (so, an error occurs).

```php
<?php

// Params
$LOGIN = '<username>@hidora.com';
$PASSWORD = '<password>';
$ENV_NAME = 'my-xonotic'; // Must be unique at each execution
$INSTANCE_NAME = 'Xonotic';

$settings = '{"SERVER_NAME": "My Xonotic Server", "MAX_PLAYER": 10}';
$manifest = '{"type":"install","version":1.4,"name":"Xonotic Server","displayName":"Xonotic Server","homepage":"http://www.xonotic.org/","logo":"https://assets.gitlab-static.net/xonotic/xonotic/raw/12105b36a21e7472f72933b6dd409465b5133396/misc/logos/icons_png/xonotic_48.png","description":"Game server for Xonotic","settings":{"fields":[{"type":"string","name":"SERVER_NAME","caption":"Server name","placeholder":"Xonotic server of ${user.name}","default":"Xonotic server of ${user.name}"},{"type":"numberpicker","name":"MAX_PLAYER","caption":"Max players","placeholder":8,"default":8,"min":2,"max":32,"editable":true}]},"nodes":[{"image":"5ika/xonotic","count":1,"fixedCloudlets":4,"cloudlets":32,"nodeGroup":"cp","volumes":["/opt/Xonotic","/opt/Xonotic/data/server.cfg"]}],"onInstall":[{"cmd[cp]":["wget -O /opt/Xonotic/data/server.cfg https://raw.githubusercontent.com/HidoraSwiss/manifest-xonotic/master/server.cfg","sed -i \"s/SERVER_NAME/${settings.SERVER_NAME}/g\" /opt/Xonotic/data/server.cfg","sed -i \"s/HOSTER/${env.domain}/g\" /opt/Xonotic/data/server.cfg","sed -i \"s/MAX_PLAYER/${settings.MAX_PLAYER}/g\" /opt/Xonotic/data/server.cfg"]},{"script":{"script":"https://raw.githubusercontent.com/HidoraSwiss/manifest-utilities/master/scripts/addEndpoint.js","params":{"nodeId":"${nodes.cp.first.id}","protocol":"UDP","port":26000,"name":"Xonotic server","text":"Use this address to connect to your Xonotic server :"},"type":"js"}}],"success":"# Your Xonotic server is ready !\nThe information for connection has been sent to you by e-mail\n\n## Configuration\nIf the default configuration doesn\'t suit you, you can modify the config file `/opt/Xonotic/data/server.cfg` and restart the Xonotic node.\n"}';

// Send a request to the Hidora API
function request($url, $data) {
  $options = array(
    'http' => array(
        'header'  => "Content-type: application/x-www-form-urlencoded\r\n",
        'method'  => 'POST',
        'content' => http_build_query($data)
      )
  );
  $context  = stream_context_create($options);
  echo("Request sent to ".$url."<br/>");
  $result = file_get_contents($url, false, $context);
  if ($result === FALSE) { 
    var_dump($result);
  }
  return json_decode($result, true);
}

// Get a session ID
$session = request('https://app.hidora.com/1.0/users/authentication/rest/signin', 
                   array('appid' => '1dd8d191d38fff45e62564fcf67fdcd6', 'login' => $LOGIN, 'password' => $PASSWORD))['session'];

// Run the manifest on Hidora
$response = request('https://app.hidora.com/1.0/development/scripting/rest/eval',
                   array(
                   	'script' => 'InstallApp',
                    'session' => $session,
                    'appid' => 'appstore',
                    'shortdomain' => $ENV_NAME,
                    'envName' => $ENV_NAME,
                    'type' => 'install',
                    'name' => $INSTANCE_NAME,
                    'displayName' => $INSTANCE_NAME,
                    'region' => 'vz7',
                   	'settings' => $settings,
                    'manifest' => $manifest
                   ));

// Close session
request('https://app.hidora.com/1.0/users/authentication/rest/signout', 
       array('appid' => '1dd8d191d38fff45e62564fcf67fdcd6', 'session' => $session));

echo("<br/>Environnement created");
?>

```
