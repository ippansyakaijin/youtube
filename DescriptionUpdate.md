動画の説明を一括更新<br><br>
スクリプトはまだいじると思います。<br>
更新したらYouTubeチャンネルのコミュニティにアップします

```
const ERROR_PLAYLIST_NOTFOUND = '入力されたIDから再生リストが取得できませんでした。IDが間違っていないか、取得権限のあるアカウントか確認してください';
const ERROR_HEADER_EMPTYTEXT = 'ヘッダーが空欄です。動画の説明修正用シートを確認してください';
const ERROR_FOOTER_EMPTYTEXT = 'フッターが空欄です。動画の説明修正用シートを確認してください';
const ERROR_UPDATELIMIT_DESCRIPTION = 'アップロードする動画の最大に達しています。最大値を変更するかアップロードする動画の数を調整してください。';

const PLAYLIST_SHEETNAME = '動画リスト';
const UPDATEDESCRIPTION_SHEETNAME = '動画の説明修正用';

const INPUT_WINDOWTITLE = '取得する再生リストIDを入力してださい';
const INPUT_WINDOWPROMPT = '再生リストIDは、再生リストのURLの部分にあるのでそれをコピペしてください→例）PLMApGWH6wCL9oyZzNlWbqpNeA9qm7zPhV';

const HEADER_ROW = 1;
const FOOTER_ROW = 2;
const NEWLINE_ROW = 3;
const INSERTBODY_ROW = 4;

const DESCRIPTION_ARRAY_INDEX = 9;
const DESCRIPTION_COLUMN_INDEX = 10;

const UPDATE_LIMIT = 100;
const PLAYLISTPART = 'snippet';
const VIDEOPART = 'snippet, status, contentDetails, statistics';

// ======================================================

function onOpen() {
  var ui = SpreadsheetApp.getUi(); // スプレッドシートのUIを取得
  // メニューに項目を追加
  ui.createMenu('YouTubeコマンド') // メニュー名
    .addItem('プレイリストIDから動画を取得して動画リストシートに出力（YouTubeコスト使用）', 'getVideosFromPlaylist') // 
    .addItem('シートにある全ての動画説明文から古い冒頭定型文を削除', 'removeOldText') // 
    .addItem('シートにある全ての動画説明文に冒頭定型文を追加する', 'insertNewText') // 
    .addItem('シートの動画説明文をYouTubeに反映（YouTubeコスト使用）', 'updateDescriptionAll') // 
    .addToUi(); // UIに追加
}

// ======================================================

// プレイリストIDから動画を取得して動画リストシートに出力
function getVideosFromPlaylist(){
  const playListId = inputPlaylistID_();  
  let array = [];
  // プレイリストに含まれる動画を配列に格納
  addTableIndexName_(array);
  extractVideosToArray_(playListId, array);  
  // 配列に動画の情報が入っていなければ中断
  if(array.length < 2) return;
  // シートに出力
  outputArrayToVideosListSheet_(array);

}

// 古いテキストの削除
function removeOldText(){
  const descriptionItems = convertBColumnTo1D_(getSheetAsArray_(UPDATEDESCRIPTION_SHEETNAME));
  // どっちか１つでも空欄なら終了
  if(isHeaderTextEmpty_(getHeaderText_(descriptionItems)) || isFooterTextEmpty_(getFooterText_(descriptionItems))) return;
  // 修正したDescriptionを動画リストに反映
  editDescription_(PLAYLIST_SHEETNAME, removeAndConvertTo2DArray_(getSheetAsArray_(PLAYLIST_SHEETNAME), descriptionItems));
}

// 新しいテキストの追加
function insertNewText(){
  const descriptionItems = convertBColumnTo1D_(getSheetAsArray_(UPDATEDESCRIPTION_SHEETNAME));
  // どっちか１つでも空欄なら終了
  if(isHeaderTextEmpty_(getHeaderText_(descriptionItems)) || isFooterTextEmpty_(getFooterText_(descriptionItems))) return;
  // 修正したDescriptionを動画リストに反映
  editDescription_(PLAYLIST_SHEETNAME, insertAndConvertTo2DArray_(getSheetAsArray_(PLAYLIST_SHEETNAME), createInsertText_(descriptionItems)));
}

// 動画リストシートの動画の説明をYouTubeチャンネルの動画に反映
function updateDescriptionAll(){
  const playlistTable = getSheetAsArray_(PLAYLIST_SHEETNAME);
  
  // 最大値チェック
  if(checkUpdateVideoLimit_(playlistTable.length)) return;
  
  for(let i = 1; i < playlistTable.length; i++){

    // 空欄の場合コストが勿体ないのでこの時点でスキップ
    let videoId = getVideoIdFromTable_(playlistTable,i);
    if(checkEmptyVideoIdByIndex_(videoId, i)) continue; 

    // 指定したIDの動画が見つからない（間違えている、更新権限がないなど）場合はスキップ
    let videoItem = getVideoItem_(videoId);
    if(checkEmptyVideoItem_(videoItem)) continue;

    updateVideoDescription_(videoItem, getDescriptionFromTable_(playlistTable, i));
  }
}

// ======================================================

// 名前を指定してシートを取得する
function getSheet_(name){
  return SpreadsheetApp.getActiveSpreadsheet().getSheetByName(name);
}

// シートの情報を配列に格納
function getSheetAsArray_(sheetName){
  const sh = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(sheetName);
  return sh.getRange(1,1,sh.getLastRow(),sh.getLastColumn()).getValues();
}

// 2次元配列からB列を1次元配列に変換する
function convertBColumnTo1D_(table){
  return table.map(function(row){return row[1]});
}

// 1次元配列を2次元配列に変換する
function convertTo2DArray_(items){
  return items.map(item => [item]);
}

// プレイリスト入力
function inputPlaylistID_(){
  return Browser.inputBox(INPUT_WINDOWTITLE, INPUT_WINDOWPROMPT, Browser.Buttons.OK);
}

// 動画リストシートに出力する
function outputArrayToVideosListSheet_(table){
  const sh = getSheet_(PLAYLIST_SHEETNAME);
  // 動画リストシートのクリア
  sh.clear;
  // 出力
  sh.getRange(1, 1, table.length, table[0].length).setValues(table);
  // 例外処理：垂直方向を上にする
  sh.getRange(1, 1, table.length, table[0].length).setVerticalAlignment('top');
}

// 動画を抽出して配列に格納する
function extractVideosToArray_(id, table){  
  let playListItems = '';
  let nextPageToken = '';
  
  do{
    playListItems = getPlaylist_(id, nextPageToken);
    if(checkEmptyPlaylist_(playListItems)) break;

    for(const pl of playListItems.items){
      // 動画情報
      addTableVideoInfo_(pl, table);
    }
    nextPageToken = playListItems.nextPageToken || '';

  }while(nextPageToken && UPDATE_LIMIT > table.length);
}

// 動画のインデックス名を設定する
function addTableIndexName_(table){
  table.push(['id', 'サムネurl', 'title', '公開設定', '日付', '再生数', 'コメント', 'いいね', '時間', '概要']);
}

// 動画の情報を取得して配列に追加する
function addTableVideoInfo_(playlistItem, table){  
  // 動画情報取得
  let videoItem = getVideoItem_(getVideoId_(playlistItem));

  let items = [];
  items[0] = getVideoId_(playlistItem);
  items[1] = getVideoThumbnailUrl_(videoItem);
  items[2] = getVideoTitle_(videoItem);
  items[3] = getVideoPrivacy_(videoItem);
  items[4] = getVideoDate_(videoItem);
  items[5] = getVideoViewCount_(videoItem);
  items[6] = getVideoCommentCount_(videoItem);
  items[7] = getVideoLikes_(videoItem);
  items[8] = getVideoTime_(videoItem);
  items[9] = getVideoDescription_(videoItem);
  Logger.log('No ' + table.length + ':' + getVideoTitle_(videoItem));

  table.push(items);
}

// プレイリストが無ければtrueを返す
function checkEmptyPlaylist_(pl){
  if(!pl){
    Logger.log('> 指定したplaylistが見つかりませんでした');
    return true;
  }
  return false;
}

// プレイリスト取得
function getPlaylist_(id, token){
  return YouTube.PlaylistItems.list(PLAYLISTPART, {'playlistId': id, 'maxResults': 10, 'pageToken': token});
}

// 動画のID取得
function getVideoId_(playlistItem){
  return playlistItem.snippet.resourceId.videoId;
}

// 動画のサムネイルURL取得
function getVideoThumbnailUrl_(videoItem){
  return videoItem.snippet.thumbnails.default.url;
}

// 動画のタイトル取得
function getVideoTitle_(playlistItem){
  return playlistItem.snippet.title;
}

// 動画の公開範囲取得
function getVideoPrivacy_(videoItem){
  return videoItem.status.privacyStatus;
}

// 動画の日付取得
function getVideoDate_(videoItem){
  return convertISO8601ToDate_(videoItem.snippet.publishedAt);
}

// 動画の再生数取得
function getVideoViewCount_(videoItem){
  return videoItem.statistics.viewCount;
}

// 動画のコメント数取得
function getVideoCommentCount_(videoItem){
  return videoItem.statistics.commentCount;
}

// 動画のいいね数取得
function getVideoLikes_(videoItem){
  return videoItem.statistics.likeCount;
}

// 動画の再生時間取得
function getVideoTime_(videoItem){
  return convertISO8601ToTime_(videoItem.contentDetails.duration);
}

// 動画の説明文取得
function getVideoDescription_(videoItem){
  return videoItem.snippet.description;
}

// 再生時間をISO8601からhh:mm:ssに変換する
function convertISO8601ToTime_(iso8601){

  let matches = iso8601.match(/PT(?:(\d+)H)?(?:(\d+)M)?(?:(\d+)S)?/);

  let hours = isTimeFromISO8601_(matches[1]);
  let minutes = isTimeFromISO8601_(matches[2]);
  let seconds = isTimeFromISO8601_(matches[3]);

  return [hours, minutes, seconds].join(':');
}

// 時間部分を抽出。なければ0を返す
function isTimeFromISO8601_(time){
  if(time){
    return parseInt(time, 10).toString().padStart(2, '0');
  }
  return 0;
}

// アップデートの日付を変換
function convertISO8601ToDate_(iso8601){
  let list = iso8601.split(/[-T:Z]/);
  let date = [list[0], list[1], list[2]].join('/');
  let time = [list[3], list[4], list[5]].join(':'); 

  return date + ' ' + time;
}

// 開始位置定型文の取得
function getHeaderText_(items){
  return items[HEADER_ROW];
}

// 終了位置定型文の取得
function getFooterText_(items){
  return items[FOOTER_ROW];
}

// 改行の数を取得
function getNewLineCount_(items){
  return items[NEWLINE_ROW];
}

// 空白ならtrueを返す
function isTextEmpty_(string){
  if(string === ''){
    return true;
  }
  return false;
}

// 第一引数の中に第二引数があればindexを返す
function findTextIndex_(target, text){
  return target.indexOf(text);
}

// 修正した概要欄のテキストをシートに反映する
function editDescription_(sheetName, table){
  Logger.log(table);
  const sh = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(sheetName);
  sh.getRange(1, DESCRIPTION_COLUMN_INDEX, table.length, 1).setValues(table);
}

// 動画リストの概要欄を上から順番にテキストを削除していき、2次元配列に変換して返す
function removeAndConvertTo2DArray_(table, items){
  let resultItems = [table[0][DESCRIPTION_ARRAY_INDEX]];

  for(let i = 1; i < table.length; i++){
    Logger.log(i+'回目 ' + table[i][0]);
    resultItems.push(removeUnwantedText_(table[i][DESCRIPTION_ARRAY_INDEX], getHeaderText_(items), getFooterText_(items), getNewLineCount_(items)));
  }
  return convertTo2DArray_(resultItems);
}

// 該当文言があれば概要欄から削除
function removeUnwantedText_(description, header, footer, newline){
  let headerIndex = findTextIndex_(description, header);
  let footerIndex = findTextIndex_(description, footer);

  // 該当テキストがあれば削除実行、無ければスルー
  if(headerIndex >= 0 && footerIndex >= 0){
    Logger.log('> 削除テキスト発見→削除実行');
    return description.substring(0, headerIndex) + description.substring(footerIndex + footer.length + newline);
  } else {
    Logger.log('> 該当テキストが無いのでスルー');
    return description;
  }
}

// 動画リストの上から順番にテキストの差し込み処理を実行後、最後に2次元配列にして返す
function insertAndConvertTo2DArray_(table, insertText){
  let resultItems = [table[0][DESCRIPTION_ARRAY_INDEX]];

  for(let i = 1; i < table.length; i++){
    Logger.log(i+'回目 ' + table[i][0]);
    resultItems.push(insertTextAsDescription_(table[i][DESCRIPTION_ARRAY_INDEX], insertText));
  }  
  return convertTo2DArray_(resultItems);
}

// 既にある動画の説明の冒頭に、動画の説明修正用のテキストを差し込む
function insertTextAsDescription_(description, text){
  return text + description;
}

// 追加するテキストの合成
function createInsertText_(items){
  // 定型文の最後
  let text = getHeaderText_(items) + '\n';
  // 本文合成
  for(let i = INSERTBODY_ROW; i < items.length; i++){
    text += items[i] + '\n';
  }
  // 定型文の最後と改行
  text += getFooterText_(items) + getNewLineString_(getNewLineCount_(items));

  return text;
}

// 動画の説明修正用の改行数の分だけ'\n'を増やして返す
function getNewLineString_(count)
{
  let str = '';
  for(let i = 0; i < count; i++){
    str += '\n';
  }
  return str;
}

// 動画リストのvideoidが空欄になっていないか？
function checkEmptyVideoIdByIndex_(id, index){
  // デバッグ表示用に調整
  index += 1;
  Logger.log(index + '行目 ' + id);
  if(id == ''){
    Logger.log('> ' + index + '行目のvideoIdが空欄の為、概要欄更新はスキップ');
    return true;
  } 
  return false;
}

// 指定したvideoidはyoutubeに存在しているか？
function checkEmptyVideoItem_(item){
  if(!item){
    Logger.log('> 指定したvideoidが見つかりませんでした');
    return true;
  }
  return false;
}

// 動画リストから指定したrowIndexのvideoidを取得
function getVideoIdFromTable_(table, rowIndex){
  return table[rowIndex][0];
}

// 動画リストから指定したrowIndexのDescriptionを取得
function getDescriptionFromTable_(table, rowIndex){
  return table[rowIndex][DESCRIPTION_ARRAY_INDEX];
}

// 動画の情報を取得
function getVideoItem_(id){
  return YouTube.Videos.list(VIDEOPART, {'id':id}).items[0];
}

// 動画の説明を更新する
function updateVideoDescription_(videoItem, text){
  Logger.log('> 概要欄を更新')
  videoItem.snippet.description = text;
  try{
    YouTube.Videos.update(videoItem, VIDEOPART);
  }catch(e){
    Logger.log('ERROR');
  }  
}

// ======================================================

// プレイリストIDが空だったらERROR判定
function notFoundPlayList_(){
  Browser.msgBox(ERROR_PLAYLIST_NOTFOUND);
  Logger.log(ERROR_PLAYLIST_NOTFOUND);
}

// 削除テキストのヘッダーが空欄だったらERROR判定
function isHeaderTextEmpty_(header){
  if(isTextEmpty_(header)){
    Browser.msgBox(ERROR_HEADER_EMPTYTEXT);
    Logger.log(ERROR_HEADER_EMPTYTEXT);
    return true;
  }
  return false;
}

// 削除テキストのフッターが空欄だったらERROR判定
function isFooterTextEmpty_(footer){
  if(isTextEmpty_(footer)){
    Browser.msgBox(ERROR_FOOTER_EMPTYTEXT);
    Logger.log(ERROR_FOOTER_EMPTYTEXT);
    return true;
  }
  return false;
}

// 更新する動画の数が100以上ならERROR判定
function checkUpdateVideoLimit_(numRows){
  if(numRows >= UPDATE_LIMIT){
    let text = UPDATE_LIMIT + '件：' + ERROR_UPDATELIMIT_DESCRIPTION;
    Browser.msgBox(text);
    Logger.log(text);
  }
  return false;
}




```
