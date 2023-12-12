## 5. 개발 기간 및 작업관리

### 개발 기간
* 전체 개발 기간 : 2023-11-22 ~ 2023-12-07

### 작업관리
* GitHub를 사용하여 프로젝트 협업을 진행하였습니다.
* 매일 프로젝트를 진행하기 전 작업 순서와 방향성에 대해 회의를 하고 새롭게 배운 내용을 공유하는 시간을 가졌습니다.

## 8. 트러블 슈팅
* AJAX 동시 사용 이슈
* AWS S3 사진 파일 관리
* AWS Lambda


## AJAX 동시 사용이슈

* 문제상황
비동기 통신으로 DB에서 사진의 경로를 가져오고 다시 비동기 통신으로 S3에서 사진을 가져오는 상황에서
경로를 받지 못하고 두 번째 AJAX가 실행이 되는 문제가 발생했다.

![화면 캡처 2023-12-11 164825](https://github.com/KIMGUUNI/test01/assets/142488092/6f61592d-773f-4286-955e-1e9742ac363b)


* 트러블 원인 추론
  * DB에서 사진 경로를 받고 다시 AWS에서 그 경로로 사진 파일 가져오는 과정
<pre><code> 
  // DB에서 사진경로 받는 기능
  function rank() {
	var Groupno = new Array;
	var StringNo = null;
		$.ajax({
			//data : sendObj,
			url: "http://localhost:8081/YOU_I/rank.do",
			dataType: "json",
			success: function(res) {
				var i = 0;
				StringNo = "";
				res.forEach(function(res) {
					var temp = res.groupNo
					StringNo += temp + ",";
					Groupno.push(StringNo);
					$(".groupinfo" + i).after("<tr><td class='grouptd'>" + res.groupInfo + "</td></tr>");
					$(".groupinfo" + i).after("<tr><td class='grouptd'>" + res.groupName + "</td></tr>");
					$(".groupinfo" + i).after("<tr><td class='grouptd'>" + res.hobbyName + "</td></tr>");
					i++;
					return Groupno;
				});
			},
			error: function(e) {
			}
		})
}
var Group_no = rank()

  //S3에서 사진 가져오는 기능
	var Groupnodata = { data: Group_no }
 	var bucketUrl = "https://s3.ap-northeast-2.amazonaws.com/you-i/"
	$.ajax({

		url: "http://localhost:8081/YOU_I/GroupImageTake.do",
		data: Groupnodata,
		dataType: "json",
		success: function(res) {
			var i = 1;
			res.forEach(function(res){
			console.log("https://s3.ap-northeast-2.amazonaws.com/you-i/resize_profile"+res.fileThumb)
			var imagePath = "resize_profile"+res.fileThumb;
			var finalUrl = bucketUrl + encodeURIComponent(imagePath);
				$("#groupimg"+i).append(`<img id='groupThum' src=${finalUrl}>`);
				i++;
			})
		},
		error: function(e) {
				console.log(e);
			}
	}) </code></pre>



* 해결방법
  * Promise를 사용하여 rank() 함수에서 경로를 받을 때까지 대기한 다음 두 번째 AJAX를 실행하게 했다.
 

 <pre><code> 
// DB에서 사진경로 받는 기능
function rank() {
	var Groupno = new Array;
	var StringNo = null;
	return new Promise(function(resolve, reject) {
		$.ajax({
			//data : sendObj,
			url: "rank.do",
			dataType: "json",
			success: function(res) {
				var i = 0;
				StringNo = "";
				res.forEach(function(res) {
					var temp = res.groupNo
					StringNo += temp + ",";
					Groupno.push(StringNo);
					console.log(Groupno);
					$(".groupinfo" + i).after("<tr><td class='grouptd'>" + res.groupInfo + "</td></tr>");
					$(".groupinfo" + i).after("<tr><td class='grouptd'>" + res.groupName + "</td></tr>");
					$(".groupinfo" + i).after("<tr><td class='grouptd'>" + flaticonHobby(res.hobbyNo) + res.hobbyName + "</td></tr>");
					i++;
				});
					resolve(StringNo);
			},
			error: function(e) {
				console.log(e);
			}
		})
	})
}
 //S3에서 사진 가져오는 기능
rank().then(function(resolvedData) {
	console.log(resolvedData);
	var Groupnodata = { data: resolvedData }
 	var bucketUrl = "https://s3.ap-northeast-2.amazonaws.com/you-i/"
	$.ajax({

		url: "GroupImageTake.do",
		data: Groupnodata,
		dataType: "json",
		success: function(res) {
			var i = 1;
			res.forEach(function(res){
			console.log("https://s3.ap-northeast-2.amazonaws.com/you-i/resize_profile"+res.fileThumb)
			var imagePath = "resize_profile"+res.fileThumb;
			var finalUrl = bucketUrl + encodeURIComponent(imagePath);
				$("#groupimg"+i).append(`<img src=${finalUrl}>`);
				i++;
			})
		},
		error: function(e) {
				console.log(e);
			}
	})
})
 </code></pre>

 * 해결된 사진

   
![화면 캡처 2023-12-11 165043](https://github.com/KIMGUUNI/test01/assets/142488092/3dcf92a0-f30f-410d-93df-d7c3db2953fb)

   


## AWS S3 사진관리 문제

* 문제상황
	* 사진을 대량으로 업로드 되는 커뮤니티 사이트의 사진을 관리하는 방법이 필요했다. 또 이름이 중복되어있는 사진을 저장했을 때 덮어 씌워지는 문제가 발생했다.

  ![화면 캡처 2023-12-07 190706](https://github.com/KIMGUUNI/test01/assets/142488092/bbc10616-06d8-4c7f-b466-ff5c19260a44)


  * 문제 코드
  	 *  S3의 경로에 저장하는 코드
   
      
 <pre><code> 
	//사진 저장하는 기능
 const upload = new AWS.S3.ManagedUpload({
			params: {
				Bucket: albumBucketName,
				Key: file.name,
				Body: file,
				ACL: "public-read"
			}
		});
 </code></pre>

* 문제해결
	* 그룹마다 각각 폴더에 저장하는 경로를 만들고 사진 이름이 중복되지 않게 범용 고유 식별자 UUID를 사용해서 사진을 관리했다.
 

<pre><code>

	let randomRoot = guid();
// 사진 저장하는 기능
		const upload = new AWS.S3.ManagedUpload({
			params: {
				Bucket: albumBucketName,
				Key: 'images/Group' + fileS3Root+ '/' + randomRoot + '_' + year + '_' + month + '_' + day + '_' + file.name,
				Body: file,
				ACL: "public-read"
			}
		});

	
// UUID 생성함수
	function guid() {
		function s4() {
			return ((1 + Math.random()) * 0x10000 | 0).toString(16).substring(1);
		}
		return s4() + s4() + '-' + s4() + '-' + s4() + '-' + s4() + '-' + s4() + s4() + s4();
	}
</code></pre>

* 해결된 사진

  
 ![화면 캡처 2023-12-07 190159](https://github.com/KIMGUUNI/test01/assets/142488092/f0f2cdc9-36c6-472b-8652-1c1e09ad22a7)



## AWS Lambda

* 커뮤니티 페이지 속도를 높히기 위해 원본이 아닌 리사이징 된 사진을 저장하고 사용해야 한다. 그렇기 떄문에 S3에 원본사진과 리사이징된 사진을 동시에 저장을 해야했고, 업로드 요청을 두번 보내야 했기 때문에 비효율적인 호출이 발생하는 문제가 생겼다.

* 문제해결
	*  AWS에서 제공하는 람다 함수를 사용해서 S3에 사진을 저장하면 이벤트 알람을 호출해 자동으로 리사이징해서 저장할 수 있게 했다.

<pre><code>

//사진 리사이징 함수
def resize_image(image_path, resized_path):
  with Image.open(image_path) as image:
    image.thumbnail(tuple(x / 2 for x in image.size))
    image.save(resized_path, quality=30)

// S3에 리사이징 된 이미지를 저장하는 함수
def lambda_handler(event, context):
  for record in event['Records']:
    bucket = record['s3']['bucket']['name']
    key = unquote_plus(record['s3']['object']['key'])
    print(bucket)
    print(key)
    tmpkey = key.replace('/', '')
    print(tmpkey)
    download_path = '/tmp/{}{}'.format(uuid.uuid4(), tmpkey)
    print(download_path)
    upload_path = '/tmp/resized-{}'.format(tmpkey)
    print(upload_path)
    print('resize_{}'.format(key))
    s3_client.download_file(bucket, key, download_path)
    resize_image(download_path, upload_path)
    s3_client.upload_file(upload_path, bucket, 'resize_{}'.format(key))

</code></pre>

* 해결된 사진

  ![화면 캡처 2023-12-12 150016](https://github.com/KIMGUUNI/test01/assets/142488092/0c831683-1ba7-496f-bac4-bc00cd2d7fec)

