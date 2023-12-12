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

   

