# youtube
youtubeで使ったスクリプトの格納庫<br>
初めてGitHub使ったけどこれであってるのか…？<br>
<br>
<br>
チャンネルIDから動画一覧を取得するスクリプト
```
function myFunction() {

  // 初期設定 =======================================
  // 調べたいチャンネルID
  const channelId = 'CHANNEL_ID'; 
  // 取得する動画の最大値（大きすぎる値はタイムアウトになります）
  const limit = 2;
  // ===============================================


  // 出力するスプレッドシート取得
  const sh = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('シート1');
  sh.clear();

  // チャンネル取得
  const channelPart = 'contentDetails, snippet, status';
  const channel = YouTube.Channels.list(channelPart, {'id': channelId});
  
  // 取得するプレイリストのパラメータ
  const playListId = channel.items[0].contentDetails.relatedPlaylists.uploads;
  const playListPart = 'snippet';

  // 取得する動画のパラメータ
  const videoPart = 'snippet, statistics, contentDetails, status';
  let nextPageToken = '';

  // スプレッドシート出力用配列
  let array = [];
  array.push(['id', 'サムネurl', 'title', '公開設定', '日付', '再生数', 'コメント', 'いいね', '時間']);

  do{
    // プレイリスト取得
    let playListItems = YouTube.PlaylistItems.list(playListPart, {'playlistId': playListId, 'maxResults': 1, 'pageToken': nextPageToken});

    for(const pl of playListItems.items){

      // 動画情報取得
      let videoId = pl.snippet.resourceId.videoId;
      videoId = '7ljvcWyYeSE';
      let video = YouTube.Videos.list(videoPart, {'id': videoId});
      let videoItem = video.items[0];

      // 出力用変数
      let thumbnailUrl = videoItem.snippet.thumbnails.maxres.url;
      let videoTitle = pl.snippet.title;
      let videoPrivacy = videoItem.status.privacyStatus;
      let videoDate = videoItem.snippet.publishedAt;
      let videoCount = videoItem.statistics.viewCount;
      let videoComment = videoItem.statistics.commentCount;
      let videoLike = videoItem.statistics.likeCount;
      let videoTime = videoItem.contentDetails.duration;

      Logger.log('No ' + array.length + ':' + videoTitle);

      // 配列に追加
      array.push([videoId, thumbnailUrl, videoTitle, videoPrivacy, videoDate, videoCount, videoComment, videoLike, videoTime]);
    }
    // 
    nextPageToken = playListItems.nextPageToken || '';

  }while(nextPageToken && limit > array.length);
  
  // スプレッドシートに出力する
  sh.getRange(1, 1, array.length, array[0].length).setValues(array);

}
```
<br>

動画IDから動画一覧を取得するスクリプト
```
function commentTest() {

  // 初期設定 =======================================
  // 調べたい動画ID
  const videoId = 'VIDEO_ID'; 
  // 取得する動画の最大値（大きすぎる値はタイムアウトになります）
  const limit = 1000;
  // ===============================================


  // 動画取得
  const videoPart = 'snippet, statistics';
  const video = YouTube.Videos.list(videoPart, {'id': videoId});

  // 取得するコメントのパラメータ
  const commentPart = 'snippet';
  let nextPageToken = '';

  // スプレッドシート出力用配列
  let array = [];
  array.push(['id', '名前', '日付', 'いいね', 'コメント']);

  do{
    // コメントスレッド取得
    let commentThreads = YouTube.CommentThreads.list(commentPart, {'videoId': videoId, 'maxResults': 100, 'pageToken': nextPageToken});

    for(const ct of commentThreads.items){
      
      let textList = YouTube.Comments.list(commentPart,{'id': ct.id});
      let textItem = textList.items[0];

      let name = textItem.snippet.authorDisplayName;
      let text = textItem.snippet.textDisplay;
      let date = textItem.snippet.publishedAt;
      let like = textItem.snippet.likeCount;

      Logger.log('No ' + array.length + ':' + name);

      // 配列に追加
      array.push([ct.id, name, date, like, text]);

      // 返信があれば返す
      let response = YouTube.Comments.list(commentPart, {'parentId': ct.id});

      if(response.items.length != 0)
      {
        for(const res of response.items){
          name = res.snippet.authorDisplayName;
          text = res.snippet.textDisplay;
          date = res.snippet.publishedAt;
          like = res.snippet.likeCount;

          Logger.log('No ' + array.length + ':' + name);
          
          // コメントの返信を追加
          array.push(['>>', name, date, like, text]);
        }
      }
    }

    nextPageToken = commentThreads.nextPageToken || '';

  }while(nextPageToken && limit > array.length);

  // 出力用のシート追加
  const sh = SpreadsheetApp.getActiveSpreadsheet().insertSheet('comment'+ new Date());

  // 取得した動画タイトルを出力
  sh.getRange(1, 1).setValue(video.items[0].snippet.title);
  // スプレッドシートにコメント出力
  sh.getRange(2, 1, array.length, array[0].length).setValues(array);

}
```
