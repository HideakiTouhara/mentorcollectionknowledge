MentorCollection制作Knowledgeari
MentorCollection制作Knowledge

*TECH::CAMP VRコースの研修用アプリです。*
*開発環境 Unity5.5.1f以降*


## 仕様の確認
基本はAbyssRiumの様な、画面をタップするとお金が溜まっていき、どんどんお金を増やしていくゲームです。
メンターを雇うと1タップあたりの生産性が上がります。
メンターのレベルを上げると1タップあたりの生産性が上がります。
あと1秒ごとに1タップあたりの生産性/2を自動的に加算します。
実際やってみる方が早いでしょう。
たくまくんに、自分のスマートフォンにビルドしてもらいましょう。

箇条書きで要素挙げます。
- CSVを読み込んでデータをクラスにマッピング
- マスターデータから各種データの算出
- UIの組み立て
- 拡張メソッド
- UniRx
- iOS/Android Portrate/Landscape
- Google Cardboard
- JsonにしてPlayerPrefsにデータSave/Load


## データテーブル
スプレッドシートで以下の様なデータベースを作成していきます。
[__こちら__](https://docs.google.com/spreadsheets/d/1mYUT577B26EFcw9ifaXWbMMoUHvbkR_NyHYdh3dc94k/edit#gid=726592699)のスプレッドシートを参照してくだおっぱい
characterっていうシートを使います。
編集したい場合は自分でシート作ってコピーしてください。

ちなみにこのようなゲーム内で使う基盤となるデータを __マスターデータ__ と呼びます。
これわかりやすいかも(?) -> http://labs.gree.jp/blog/2015/12/15368/

今回のマスターデータの運用は、スプレッドシートから直接取り込むということにします。

該当するシートを開いた状態で、
File > Download as > Comma-separated values (.csv, current sheet)
でダウンロードできます。

![csv_download.png](https://qiita-image-store.s3.amazonaws.com/0/25374/aacc4fb6-8fb6-df6a-9895-ab4a276f101d.png "csv_download.png")


ダウンロードしたファイルを Character.csv という名前にしてUnityに入れておきましょう。
フォルダ分けもしておきましょうか。下記ディレクトリで。
MyAssets/Resources/CSV/Character.csv


##CSVをUnityで読み込む
[__こちら__](http://wiki.unity3d.com/index.php?title=CSVReader)の CSVReader.cs を拝借しましょう。
CSVReader.SplitCsvGlid(csvText) にcsvを渡すとstringの二次元配列で返ってくるのでそれをクラスにマッピングします。

※二次元配列は、行列をイメージすると分かりやすい。
→http://ufcpp.net/study/csharp/st_array.html

さて、まずマッピングするクラスを作りましょう。
__MstCharacter.cs__ を作成し、中身をこんなかんじにします。

MstCharacter.cs (新規)

```C#
using UnityEngine;

[System.SerializableAttribute]
public class MstCharacter
{
	[SerializeField]
	private int
		id,
		rarity,
		maxLebel,
		growthType,
		lowerEnergy,
		upperEnergy,
		initialCost;
	
	[SerializeField]
	private string
		name,
		imageId,
		flavorText;

	public void SetFromCSV(string[] data)
	{
		id 			= int.Parse( data[0] );
		name		= data[1];
		imageId		= data[2];
		flavorText 	= data[3];
		rarity		= int.Parse( data[4] );
		maxLebel	= int.Parse( data[5] );
		growthType	= int.Parse( data[6] );
		lowerEnergy = int.Parse( data[7] );
		upperEnergy = int.Parse( data[8] );
		initialCost = int.Parse( data[9] );
	}
}
```


読み込む側はこんなかんじで。マスターデータを管理するので
__MasterDataManager.cs__ でどうでしょう。

MasterDataManager.cs (新規)

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MasterDataManager : MonoBehaviour
{
	[SerializeField]
	private List<MstCharacter> characterTable = new List<MstCharacter>();

	private void Start()
	{
		var characterCSV = Resources.Load("CSV/Character.csv") as TextAsset;
		var csv = CSVReader.SplitCsvGrid(characterCSV.text);
		for (int i=1; i<csv.GetLength(1)-1; i++) 
		{
			var data = new MstCharacter();
			data.SetFromCSV( GetRaw(csv, i) );
			characterTable.Add(data);
		}
	}

	private string[] GetRaw (string[,] csv, int row) {
		string[] data = new string[ csv.GetLength(0) ];
		for (int i=0; i<csv.GetLength(0); i++) {
			data[i] = csv[i, row];
		}
		return data;
	}
}
```

※　csv.GetLength(0)で最初の次元の長さを返します。
csv.GetLength(1)で2つ目の次元の長さを返します。
(この場合は、10と15になるはずです。しかし、原因がわからないのですが、12と16になってしまいます。)

Unityに戻って空のゲームオブジェクトにMasterDataManagerをくっつけて実行すると
InspectorでcharacterTableの中にデータがマッピングされてるのがわかるはずです。

こんな感じに。
![Screen Shot 2017-03-20 at 17.42.55.png](https://qiita-image-store.s3.amazonaws.com/0/25374/a4bda225-5a0c-dfc0-a059-d76cf0fbd3d2.png "Screen Shot 2017-03-20 at 17.42.55.png")



ここでよく起きるエラーは、int.Parseしようとしてるけどパースできないよー！というエラーです。
そういうときは数字が入るべきところに全角やスペースが入っていないかスプレッドシートで確認しましょう。あと、

[System.SerializableAttribute]

をクラスの前につけないとインスペクターで見えるようになりません。
中身のデータがちゃんと入ってるか視認したいときとかにつけましょう。

## MstCharacterがprivateな変数しかないのでReadOnlyなパラメータを作る
(プロパティ、{ get; set; } を使ったことある前提で書いてるけど必要だったらあとで追記します)
get; しか書かないことで、他のクラスからは読み取りしかできない変数を作れます。

MstCharacter.cs (編集)

```C#
using UnityEngine;

[System.SerializableAttribute]
public class MstCharacter
{
	[SerializeField]
	private int
		id,
		rarity,
		maxLebel,
		growthType,
		lowerEnergy,
		upperEnergy,
		initialCost;
	
	[SerializeField]
	private string
		name,
		imageId,
		flavorText;

	public void SetFromCSV(string[] data)
	{
		id 			= int.Parse( data[0] );
		name		= data[1];
		imageId		= data[2];
		flavorText 	= data[3];
		rarity		= int.Parse( data[4] );
		maxLebel	= int.Parse( data[5] );
		growthType	= int.Parse( data[6] );
		lowerEnergy = int.Parse( data[7] );
		upperEnergy = int.Parse( data[8] );
		initialCost = int.Parse( data[9] );
	}
	
	// ここから追記部分
	public int ID 		{ get{ return id; } }
	public int Rarity 	{ get{ return rarity; } }
	public int MaxLevel { get{ return maxLebel; } }
	public int GrowthType 	{ get{ return growthType; } }
	public int LowerEnergy 	{ get{ return lowerEnergy; } }
	public int UpperEnergy 	{ get{ return upperEnergy; } }
	public int InitialCost 	{ get{ return initialCost; } }
	public string Name { get{ return name; } }
	public string ImageId { get{ return imageId; } }
	public string FlavorText { get{ return flavorText; } }
	
	// ついでに現在購入可能な状態か返す関数とか作っとく
	public bool PurchaseAvailable(int currentMoney)
	{
		return (currentMoney < InitialCost) ? false : true;
	}
}
```

ちゃんと補完が聞くエディタ使ってたら
他のクラスの関数内で

```
var chara = new MstCharacter();
chara. 
```

って書くとちゃんといま追記したやつしか出ないと思うから試してみて
でもこの教材やる人はもうその辺は大丈夫か(?)


## ローカルからCSV読み込んでるところをスプレッドシートから直接ダウンロードしてみる。
いちいちデータを変更してUnityにインポートし直すの面倒なのでスプレッドシートから動的にデータを落とすようにしてみる。
こうすることで、レベルデザインをやるプランナーの作業とエンジニアの作業が綺麗に切り分けられる。
実際にはゲームごとにサーバー側のプログラムを書いて、JSONでやり取りすることが多いが、今回はCSVで。

まずはスプレッドシートから直接CSVでダウンロードできるURLを取得する。
![csv_publish_1.png](https://qiita-image-store.s3.amazonaws.com/0/25374/8b6558c7-83da-ac55-78dc-c02c98b1cd2e.png "csv_publish_1.png")
![csv_publish_2.png](https://qiita-image-store.s3.amazonaws.com/0/25374/8c48ba2b-d6c5-eb29-7f9e-71ddcbb90bd1.png "csv_publish_2.png")



MasterDataManagerをSingletonに変更し、
Start関数をごっそりLoadData()にしてGameManagerから呼んでもらうようにする。

SingletonMonoBehaviour.cs をプロジェクト内に入れといてください
https://www.dropbox.com/s/fk1jw4o56e5ydk1/SingletonMonoBehaviour.cs?dl=0

※インスタンスを1つしか生成したくない時にSingletonを使います。
SingletonMonoBehaviourを見てみてもらうとMonoBehaviourをさらに継承しているので、これはMonoBehaviourの機能をすべて引き継いだクラスってことになります。
SingletonMonoBehaviourに
public static T instance っていうのがあるので
このクラスを継承したGameManagerとかは自分の中身にpublic static GameManager instance;　とか書かなくても親クラスに定義済みなのでそのままつかえるという感じです。

ゲームのセットアップ、Start関数はGameManagerのみ行うようにし、GameManagerが他のManager系コンポーネントのセットアップ関数を呼ぶような設計にする。
そうすることで、処理の順番がしっかり頭で把握しやすい設計になる。

まず、通信部分が使いやすくなるような通信を管理してくれるクラスを作っときましょう。

ConnectionManager.cs (新規)

```C#
using UnityEngine;
using UnityEngine.Events;
using System.Collections;
using UnityEngine.Networking;

public class ConnectionManager 
: SingletonMonoBehaviour<ConnectionManager> {

    public void ConnectionAPI (
        string url,
        UnityAction<string> onFinish = null,
        UnityAction<string> errorFinish = null
    ) {
        StartCoroutine(ConnectionAPICoroutine(url, onFinish, errorFinish));
    }

    private IEnumerator ConnectionAPICoroutine (
        string url,
        UnityAction<string> onFinish = null,
        UnityAction<string> errorFinish = null
    ) {
        var request = UnityWebRequest.Get(url);
        GameManager.Log("通信開始 : " + url);
        yield return request.Send();
        if (!request.isError) {
            GameManager.Log( "url:"+ url + "\nSuccess : " + request.downloadHandler.text );
            if (onFinish != null) {
                onFinish( request.downloadHandler.text );
            }
        }
        else {
            GameManager.Log("url:"+ url + "\nFaild : " + request.error);
            if (errorFinish != null) {
                errorFinish( request.error );
            }
        }
    }
}
```

◉初めて出てきたかもしれないやつ解説

___関数の引数に初期値を入れておくと、関数呼び出し時に引数を渡さなくてもよくなる。___
上記のスクリプトだと 
*onFinish = null, errorFinish = null*
と初期値にnullを入れているので、呼び出す側では第一引数である url だけ入れてあげれば大丈夫になります。
※（引数を書いた場合は、引数はその値になる）

```
ConnectionManager.instance.ConnectionAPI("url");
```



MasterDataManager.cs (編集)

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;
using System.Linq;

public class MasterDataManager 
: SingletonMonoBehaviour<MasterDataManager>
{
    [SerializeField]
	private List<MstCharacter> characterTable = new List<MstCharacter>();
	public List<MstCharacter> CharacterTable { get { return characterTable; } }
	// キャラクターをIDで引っ張れるようにしておく
	public MstCharacter GetCharacterById(int id)
	{
		return characterTable.Find(c => c.ID == id);
	}

	// PublishしてゲットしたURL
	const string csvUrl = "https://docs.google.com/spreadsheets/d/1mYUT577B26EFcw9ifaXWbMMoUHvbkR_NyHYdh3dc94k/pub?gid=605974578&single=true&output=csv";

    // GameManagerから呼んでもらう
	public void LoadData(UnityAction onFinish)
	{
		ConnectionManager.instance.ConnectionAPI(
			csvUrl, 
			(string result) => {
				var csv = CSVReader.SplitCsvGrid(result);
				for (int i=1; i<csv.GetLength(1)-1; i++) 
				{
					var data = new MstCharacter();
					data.SetFromCSV( GetRaw(csv, i) );
					characterTable.Add(data);
				}
				onFinish();
			}
		);
	}

    private string[] GetRaw (string[,] csv, int row) {
        string[] data = new string[ csv.GetLength(0) ];
        for (int i=0; i<csv.GetLength(0); i++) {
            data[i] = csv[i, row];
        }
        return data;
    }
}
```

GameManager.cs (新規)

```C#
using UnityEngine;
using System.Collections;

public class GameManager
: SingletonMonoBehaviour<GameManager>
{
    private void Start()
    {
        MasterDataManager.instance.LoadData(() => 
        {
            print("ロード終わったお");
        });
    }
    
    public static void Log (object log)
    {
        if (Debug.isDebugBuild)
        {
            Debug.Log(log);
        }
    }
}
```

スクリプトを書いたら空のゲームオブジェクトを作って
それぞれのコンポーネント名と同じ名前にしておくと見易いね
こんな感じにしよう↓
![Screen_Shot_2017-04-21_at_11_26_45.png](https://qiita-image-store.s3.amazonaws.com/0/25374/d2cf4674-49ea-e82d-3606-1ce0d12045e9.png "Screen_Shot_2017-04-21_at_11_26_45.png")


今回はめんどくさいのでMasterDataManagerでローカルから読み込んでたところを
通信して取ってくるように全部書き換えちゃったけど
bool loadFromLocal; みたいなの作って
LoadData()の処理をローカルから読み込むか通信して読み込むか処理分けておけば
回線ないところでも作業できるから暇だったら作っておくといいかも。

まぁ、とりあえずここまで書けたらPlayしてみて、
さっきと変わらずMasterDataManagerにデータ入ってたらOK


## UIの組み立て
まず完成イメージから確認しましょう。
UIだけに注目したいためUIだけ見せています。
[![https://gyazo.com/f9b16cb69142a3fdd0e11d708e63a1c3](https://i.gyazo.com/f9b16cb69142a3fdd0e11d708e63a1c3.gif)](https://gyazo.com/f9b16cb69142a3fdd0e11d708e63a1c3)

こんな感じで大きな塊からつくっていきましょう。
[![https://gyazo.com/ae89fd128f447ca84cc9f35f0d81ec1a](https://i.gyazo.com/ae89fd128f447ca84cc9f35f0d81ec1a.png)](https://gyazo.com/ae89fd128f447ca84cc9f35f0d81ec1a)

。
。
。
。
作成動画
[![IMAGE ALT TEXT HERE](http://img.youtube.com/vi/xvfvmKx1hUI/0.jpg)](http://www.youtube.com/watch?v=xvfvmKx1hUI)
。
。
。
。

で、このセルの部分をPrefab化してMstCharacterのデータ数分Instantiateするようにします。
[![https://gyazo.com/8df481da15fad9e67ef126a91b5e790b](https://i.gyazo.com/8df481da15fad9e67ef126a91b5e790b.png)](https://gyazo.com/8df481da15fad9e67ef126a91b5e790b)


## UIへデータをマッピングする
先ほど作ったUIに読み込んだCSVの値をマッピングします。
顔の画像使うのでこちらからダウンロードしておいて、Resourcesフォルダを作成して入れてください。
https://www.dropbox.com/sh/ly3sqcegtbdzqbd/AAD64_8JV_OfF1BtAt2tsvI6a?dl=0


__処理の順番__
CSVを読み込む → セルを配置する


まずセル一つ一つにくっつけるコンポーネントを作成します。

MentorPurchaseCell.cs (新規)

```C#
using UnityEngine;
using UnityEngine.UI;

public class MentorPurchaseCell : MonoBehaviour
{
    [SerializeField] private Image iconImage;
    [SerializeField] private Text
        nameLabel,
        rarityLabel,
        flavorTextLabel,
        productivityLabel,
        costLabel;

    [SerializeField] private Button purchaseButton;

    private bool isSold = false;
    private MstCharacter characterData;

    public void SetValue(MstCharacter data)
    {
        iconImage.sprite = Resources.Load<Sprite>("Face/" + data.ImageId);
        characterData = data;
        nameLabel.text = data.Name;
        rarityLabel.text = "";
        for (int i = 0; i < data.Rarity; i++) { rarityLabel.text += "★"; }
        flavorTextLabel.text = data.FlavorText;
        productivityLabel.text = "生産性(lv.1) : " + data.LowerEnergy;
        costLabel.text = string.Format("¥{0:#,0}", data.InitialCost);
    }
}
```

スクリプトが書けたら、Cellプレハブにコンポーネントをアタッチしたりしよう。
![Screen_Shot_2017-04-25_at_8_36_28.png](https://qiita-image-store.s3.amazonaws.com/0/25374/7d17b3ec-11e3-1190-5ef0-0ffdefd05c5e.png "Screen_Shot_2017-04-25_at_8_36_28.png")


セルができたので、Dataの配列foreachで回してSetValueに流します。

MentorPurchaseView.cs (新規)

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MentorPurchaseView : MonoBehaviour
{
	[SerializeField] private GameObject mentorPurchaseCellPrefab;
	[SerializeField] private Transform scrollContent;

	public void SetCells()
	{
		var characters = MasterDataManager.instance.CharacterTable;
		characters.ForEach(c => {
			var obj = Instantiate(mentorPurchaseCellPrefab) as GameObject;
			obj.transform.SetParentWithReset(scrollContent);
			var cell = obj.GetComponent<MentorPurchaseCell>();
			cell.SetValue(c);
		});
	}
}
```

CustomTransform.cs (新規)

```C#
using UnityEngine;

public static class CustomTransform
{
	// uGUIで、セルを並べる時に使う
	// 拡張メソッド。これについての説明はMentorPurchaseViewのスクショの下に記載
	public static Transform SetParentWithReset (this Transform self, Transform parent)
	{
		self.SetParent(parent);
		self.transform.localPosition = Vector3.zero;
		self.transform.localEulerAngles = Vector3.zero;
		self.transform.localScale = Vector3.one;
		return self;
	}
}
```

MentorPurchaseViewコンポーネントをくっつけて、スクショのようにそれぞれ紐付けます。
![Screen_Shot_2017-04-25_at_10_17_21.png](https://qiita-image-store.s3.amazonaws.com/0/25374/bda6cb86-da65-0aa0-4d0b-98f862ef41f4.png "Screen_Shot_2017-04-25_at_10_17_21.png")


### 拡張メソッドについて

```
int num = 398;
num.ToString();
```

こういうの、int型の拡張メソッドでToString()が用意されてるって感じっすね。
引数の型の宣言の前に __this__ ってつけると、使う側で . でつなぐ前の節が引数に代入されます。


ただ、関係ないクラスから勝手に拡張メソッドって作れちゃうので、
仕事で使うと嫌がられる場合も多いです。
参考 → http://d.hatena.ne.jp/yone-ken/20090304/p1

今回は明らかに Transform.SetParent() にTransformの値をリセットするっていう拡張をつけた方がスッキリするので使ってみました。
こちらからは以上です。


### なんでSetParentWithReset()っていう関数でわざわざResetしてるのかという件について
まぁ、Reset入れなかったらどうなるかっていうのを知りたければ自分の環境でこのコード入れずに実行してみて比較するっていうのがいいと思いますが一応説明しておくと、

CanvasのTransformのScale値に 0.001283634 みたいな小さい数値が入ってますよね。
親オブジェクトのスケールの値を変更すると子オブジェクトも一緒に拡大縮小してくれるっていうのと
その際、子オブジェクトのスケール値は変わらないっていうのは周知の事実だと思うんですけど、
Instantiate()で出すと当然ながらどのGameObjectの子になっていない状態でヒエラルキーに出ます。
この生成されたCellはScaleが(1,1,1)で出てきます。
もともとPrefabを作ったときはScale値が小さいCanvasの下で作っているので、これをただCanvas以下のScrollContentの子にすると、 1 / 0.001283634 = 779.0382616852(倍) のScaleに変換されてしまいます。つまりめっちゃでかい状態で表示されます。
位置とかはScrolContentにくっついてるGridLayoutGroupがなんとかしてくれますが。

何を言ってるのかよくわからなければSetParentWithReset()の
self.transform.localScale = Vector3.one;
あたりをコメントアウトしたりして再生してInspectorで見てみてください。



GameManager.cs (編集)

```C#
using UnityEngine;
using System.Collections;

public class GameManager
: SingletonMonoBehaviour<GameManager>
{
    private void Start()
    {
        MasterDataManager.instance.LoadData(() => 
        {
            print("ロード終わったお");
            // このラムダ式の部分、後で書き換えるつもりで書いてます。
            var purchaseView = GameObject.FindObjectOfType<MentorPurchaseView>();
            purchaseView.SetCells();
        });
    }
    
    public static void Log (object log)
    {
        if (Debug.isDebugBuild)
        {
            Debug.Log(log);
        }
    }
}
```	

さて、ここまでできたらCellがPlay時に生成されてくれて
データもちゃんとマッピングされてるんじゃなイカなと思います。
以上、UIへのデータのマッピングでした〜

### メンター育成するView
まだメンターを購入する画面しかないのでメンターを育成する画面を作成しておきます。
次の章のタブ切り替えで画面を切り替えられるようにします。

まず、MentorTrainingCellが保持するデータはマスターデータではなくユーザーごとに変わるキャラクターのデータなので、マスタデータをもとに生成するCharacterクラスを作成します。
levelを保持するのと、それに従って1tapあたりの生産性を返す変数を後で実装します。

Character.cs (新規)

```C#
using UnityEngine;

[System.SerializableAttribute]
public class Character 
{
	[SerializeField] private int uid, masterId, level;

	public int UniqueID { get { return uid; } }
	public int MasterId { get { return masterId; } }
	public int Level    { get { return level; } }
	
	public MstCharacter Master
	{
		get
		{
			return MasterDataManager.instance.GetCharacterById(masterId);
		}
	}
	
	public int Power 
	{
		get 
		{
			// 一旦 1 で返しておく。後で算出ロジック書く。
			return 1;
		}
	}

	// コンストラクタ
	public Character (int uniqueId, MstCharacter data)
	{
		uid = uniqueId;
		level = 1; 
		masterId = data.ID;
	}
	
	// ついで。後で使います。レベルマックスになってるかどうかを返すプロパティ
	public bool IsLevelMax 
	{ 
		get
		{  
			return (level >= Master.MaxLevel) ? true : false;
		} 
	}
}
```

uniqueっていう単語、プログラマーとかだとよく使うかもしれないけど
普通にいきてたらまぁー使わないと思うので念のため雰囲気だけ説明しておくと、
唯一の、一意の、という意味で使われます。
identifierだけだと、例えば製品の種類とかだったら同じ商品は同じIDとして扱われたりするので
データベースに重複しうる数字、文字列として扱われることが多いです。
個体を識別するIDという意味で頭にuniqueをつけて使ったりしますね。

※例えば、ポケモンでピカチュウっていうマスターデータがあって、ピカチュウが何匹でも捕まえられる。→個体IDがないと管理できない。
つまり、ピカチュウのベースのデータを管理するのがマスターデータで、捕まえて個別のデータを作る時にUniqueID(一意のID)を割り振ります。
（今回はキャラクターが被らないので、UniqueIDにする必要はないが、勉強のために使っています。）


次にUIを組みます。
MentorPurchaseViewをコピーして使おうと思うので、
一旦MentorPurchaseViewにIconと画面見出し的なやつをつけておきましょう。
![Screen_Shot_2017-05-07_at_11_12_23.png](https://qiita-image-store.s3.amazonaws.com/0/25374/36e2ea03-9ea0-503e-6625-005020249869.png "Screen_Shot_2017-05-07_at_11_12_23.png")


次に、MentorPurchaseViewをCtrl+D(⌘+D)でDuplicate(複製)し、
MentorPurhcaseViewを非表示にしておきましょう。
![Screen_Shot_2017-05-07_at_11_17_55.png](https://qiita-image-store.s3.amazonaws.com/0/25374/5021a897-2c3f-fbec-33fc-2eefd550dc50.png "Screen_Shot_2017-05-07_at_11_17_55.png")


複製したやつをMentorTrainingViewに名前を変えて見出しとIcon変えて、
くっついてるコンポーネントをMentorPurchaseViewからMentorTrainingViewに変更しちゃいましょう。
MentorTrainingViewはまだ作ってないと思うので Create > C# Script で作ってAdd Componentしましょう。
![Screen_Shot_2017-05-07_at_11_25_52.png](https://qiita-image-store.s3.amazonaws.com/0/25374/4c65bf30-bf87-f2a0-fe24-4c66d03e7402.png "Screen_Shot_2017-05-07_at_11_25_52.png")

同じようにMentorTraining用のセルもMentorPurchaseCellから複製して作りたいのでPrefabから拝借して編集していきたいと思います。
ScrollViewのContentにMentorPurchaseCellのPrefabを配置してMentorTrainingCellにRenameします。
この際、いじってすぐApplyボタンを押さないようにしましょう。
MentorPurchaseCellのPrefabを上書きしてしまいます。
[![https://gyazo.com/165b36aa9b88bb62a6893459f26422ba](https://i.gyazo.com/165b36aa9b88bb62a6893459f26422ba.png)](https://gyazo.com/165b36aa9b88bb62a6893459f26422ba)


で、MentorTrainingCell用の中身にしてPrefab化しましょう。
FlavorTextLabelをLevelLabelに変更したのと、
DescriptionButton付け加えたのと、あとVRButtonっていうのを追加しました。
あとちょこっと色をいじってます。うん、締まって見えますね。
[![https://gyazo.com/74820c296fadad7ef867280cc14c780c](https://i.gyazo.com/74820c296fadad7ef867280cc14c780c.png)](https://gyazo.com/74820c296fadad7ef867280cc14c780c)

ではMentorTrainingViewとMentorTrainingCellの中身を書いていきましょう。

MentorTrainingCell.cs (新規?)

```C#
using UnityEngine;
using UnityEngine.UI;

public class MentorTrainingCell : MonoBehaviour
{
    [SerializeField] private Image iconImage;

    [SerializeField] private Text
        nameLabel,
        rarityLabel,
        levelLabel,
        productivityLabel,
        costLabel;

    [SerializeField] private Button 
        levelUpButton, 
        discriptionButton, 
        vrButton;
    [SerializeField] private CanvasGroup levelUpButtonGroup;

    private Character characterData;

    public void SetValue(Character data)
    {
        var master = data.Master;
        characterData = data;
        iconImage.sprite = Resources.Load<Sprite>("Face/" + master.ImageId);
        nameLabel.text = master.Name;
        rarityLabel.text = "";
        for (var i = 0; i < master.Rarity; i++) { rarityLabel.text += "★"; }
        UpdateValue();
        
        levelUpButton.onClick.AddListener(() =>
        {
            // LevelUpのロジックがまだなのでまだ
        });

        discriptionButton.onClick.AddListener(() => {
            // Descriptionを表示させるところを作ってないのでまだ
        });

        vrButton.onClick.AddListener(() => {
            // VRViewを作ってないのでまだ
        });
    }

    private int CulcLevelUpCost()
    {
    	 // 一旦100円でLevelUpCostを返すようにしておく。あとで作る。
        return 100;
    }

    private void UpdateValue()
    {
        var master = characterData.Master;
        levelLabel.text = "Lv." + characterData.Level;
        productivityLabel.text = string.Format("生産性 : ¥ {0:#,0} /tap", characterData.Power);
        var cost = CulcLevelUpCost();
        costLabel.text = string.Format("¥{0:#,0}", cost);
        if (true) { //ここ、所持金がコストに足りてるかどうかっていうのに後で書き換える
        	levelUpButtonGroup.alpha = 0.5f;
        }
        else levelUpButtonGroup.alpha = 1.0f;
        if (characterData.IsLevelMax) 
        {
            levelUpButtonGroup.alpha = 0.5f;
            costLabel.text = "Level Max";
        }
    }
}
```
LevelUpButtonにCanvasGroupコーポネントを加え、アタッチします。
これ書いたら御察しの通りInspectorで紐付け作業をやる。
紐づけられるインスタンス名とGameObject名は合わせてると思うので察せられるようにできてるはず。
察せなかったらご指摘ください。謝ります。

MentorTrainingView.cs (新規?)

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class MentorTrainingView : MonoBehaviour
{
	[SerializeField] private GameObject mentorTrainingCellPrefab;
	[SerializeField] private Transform scrollContent;
	private List<MentorTrainingCell> cells = new List<MentorTrainingCell>();

	private void Start ()
	{
		gameObject.SetActive(false);
	}

	// ゲーム開始時に保存してるデータが持ってるCharacterのCellを全部生成する用
	public void SetCells()
	{
		var characters = GameManager.instance.User.Characters;
		characters.ForEach(c => { CreateCell(c); });
	}

	// ゲーム中にメンターの購入がなされたときに一つCellを追加する用
	public void AddCharacter(Character data)
	{
		CreateCell(data);
	}

	private void CreateCell(Character data)
	{
		var obj = Instantiate(mentorTrainingCellPrefab) as GameObject;
		obj.transform.SetParentWithReset(scrollContent);
		var cell = obj.GetComponent<MentorTrainingCell>();
		cell.SetValue(data);
		cells.Add(cell);
	}
}
```

こちらもSerializeFieldつけてる部分をInspectorで紐づけましょう。

### List.ForEach
foreach(var instance in array) { } とそんなに変わらないきがするけど使ってみた。
Listの要素に対してなんかしたいときにスッキリ書けると思います。
今回使ってる部分。

```
characters.ForEach(c => { CreateCell(c); });
```

下記の主張によると、continue, break が使えない、yield returnができないなど
デメリットは多いらしいけど使いたくなったらforeach使えばいいかと。
http://qiita.com/chocolamint/items/d29d699ce27bcf685154

パフォーマンスはforeachより速い、らしい。
http://devlights.hatenablog.com/entry/20120904/p1

今回においてはスッキリかけたので、まぁこんな書き方もできるよ、くらいに思っていてもいいと思います。


## タブ切り替えを実装しよう

まずはこちらをダウンロードしてインポート
https://www.dropbox.com/sh/ib3c77qp7f37mgo/AAAZLgv8UPVvyPc95zLuyruMa?dl=0

### アトラス画像からスプライトの切り出し
まずはUIに使う画像をインポートし、画像の設定を変更します。
![Screen_Shot_2017-05-02_at_9_43_02.png](https://qiita-image-store.s3.amazonaws.com/0/25374/4a9c2e42-fae4-6de4-0bfc-6b89c60b437d.png "Screen_Shot_2017-05-02_at_9_43_02.png")

次に、SpriteEditorをクリックしましょう。
![Screen_Shot_2017-05-02_at_9_58_02.png](https://qiita-image-store.s3.amazonaws.com/0/25374/8dd386d2-36e5-1dad-47ea-ebed74359e34.png "Screen_Shot_2017-05-02_at_9_58_02.png")

最後に画像をSliceします。
今回はアイコン1つあたり 128px x 128px で作っているのでこんな感じでSliceできる。
![Screen_Shot_2017-05-02_at_10_04_45.png](https://qiita-image-store.s3.amazonaws.com/0/25374/9fd737e8-b527-0cfd-f9ad-a4df510c6e75.png "Screen_Shot_2017-05-02_at_10_04_45.png")

余談ですが、
もっとたくさんアイコン作るかなって思って1024pxで画像作ってしまったけど
アイコン6こだと明らかに余白が無駄なので512pxで作った方が良い例でした
ごめんなさい。

うん、タブはアイコンがあった方がいいですね。
![Screen Shot 2017-05-02 at 18.24.55.png](https://qiita-image-store.s3.amazonaws.com/0/25374/e0f41c95-4f0d-ceee-655a-3dceb6b7380e.png "Screen Shot 2017-05-02 at 18.24.55.png")


### アトラス化が必要な理由について
画面に描画をするときにGPU的なところに画像のデータを渡すんだけど、画像がバラバラだとその渡す回数が多くなって重くなります。1枚の画像にまとめると1回で済みます。
参考
http://assetsale.hateblo.jp/entry/2015/09/02/031225

ついでにちょうど佐藤さんの記事を見つけたので貼っておく。
http://wordpress.notargs.com/blog/blog/2015/01/28/unity最適化の要「drawcall」とは？/


まぁでもモックアップ作るときとかは画像バラバラで作った方が楽なのでUnityの機能、
スプライトパッカーを使うと良いです。この制作では必要なさそうだけど。
https://docs.unity3d.com/jp/540/Manual/SpritePacker.html


### 切り替え部分のコード
下に3つのタブがあるのでタップで画面切り替えるっていうのをやります。
その前に、UI周りを管理するManagerクラスと、UIのStateのenumを作らせてください。

Const.cs (新規)

```C#
public static class Const
{
	public enum View
	{
		Purchase,
		Training,
		Close
	}
}
```

PortrateUIManager.cs (新規)

```C#
using System.Linq;
using UnityEngine;
using UnityEngine.UI;

public class PortrateUIManager
: SingletonMonoBehaviour<PortrateUIManager>
{
    [SerializeField] private MentorPurchaseView mentorPurchaseView;
    [SerializeField] private MentorTrainingView mentorTrainingView;
    public MentorPurchaseView MentorPurchaseView { get { return mentorPurchaseView; } }
    public MentorTrainingView MentorTrainingView { get { return mentorTrainingView; } }

    [SerializeField] private Text 
        moneyLabel,
        autoWorkLabel,
        employeesCountLabel,
        productivityLabel;

    [SerializeField] private Const.View 
        currentView = Const.View.Close, 
        lastView = Const.View.Purchase;

    public void Setup ()
    {
        mentorPurchaseView.SetCells();
        mentorTrainingView.SetCells();
        
        openButton.onClick.AddListener(() => {
            openButton.gameObject.SetActive(false);
            ChangeView(lastView);
        });
    }

    public void ChangeView (Const.View nextView) 
    {
        if (currentView == nextView) return;
        lastView = currentView;
        currentView = nextView;
        isMoving = true;
        switch (nextView)
        {
            case Const.View.Purchase:
                mentorPurchaseView.gameObject.SetActive(true);
                mentorTrainingView.gameObject.SetActive(false);
                break;
            case Const.View.Training:
                mentorPurchaseView.gameObject.SetActive(false);
                mentorTrainingView.gameObject.SetActive(true);
                break;
            case Const.View.Close:
                openButton.gameObject.SetActive(true);
                break;
        }
    }
    
    [SerializeField] private Button openButton;
    [SerializeField] private Transform 
        mainPanel, 
        openPoint, 
        closePoint;
    private bool isMoving = false;
    private float 
        easing = 8f,
        maxSpeed = 3f,
        stopDistance = 0.001f;

    private void Update () {
        if (!isMoving) return;
        var target = (currentView == Const.View.Close) ? closePoint : openPoint;

        // position
        Vector3 v = Vector3.Lerp(
            mainPanel.position, 
            target.position, 
            Time.deltaTime * easing) - mainPanel.position;
        if (v.magnitude > maxSpeed) v = v.normalized * maxSpeed;
        mainPanel.position += v;
        
        if (isMoving) {
            float distance = Vector3.Distance(mainPanel.position, target.position);
            if (stopDistance > distance) isMoving = false;
        }
    }
}
```

[![https://gyazo.com/ef9caaa0d7d5c41d04eae0aa3d3d34b8](https://i.gyazo.com/ef9caaa0d7d5c41d04eae0aa3d3d34b8.png)](https://gyazo.com/ef9caaa0d7d5c41d04eae0aa3d3d34b8)

TabController.cs (新規)

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;

public class TabController : MonoBehaviour {

	[SerializeField]
	private Button recluitButton, trainingButton, closeButton;

	private void Start ()
	{
		recluitButton.onClick.AddListener(() => {
			PortrateUIManager.instance.ChangeView(Const.View.Purchase);
		});
		trainingButton.onClick.AddListener(() => {
			PortrateUIManager.instance.ChangeView(Const.View.Training);
		});
		closeButton.onClick.AddListener(() => {
			PortrateUIManager.instance.ChangeView(Const.View.Close);
		});
	}
	
}
```

↑こいつはTabPanelにくっつけてボタン3つInspectorでくっつけてください。

GameManager.cs (編集)

```C#
using UnityEngine;
using System.Collections;

public class GameManager
: SingletonMonoBehaviour<GameManager>
{
    private void Start()
    {
        MasterDataManager.instance.LoadData(() => 
        {
            // ここ編集しました
            PortrateUIManager.instance.Setup();
        });
    }
    
    public static void Log (object log)
    {
        if (Debug.isDebugBuild)
        {
            Debug.Log(log);
        }
    }
}
```	

できたらUIを察して紐付けましょう。(あとでスクショ貼る)


## JsonにしてPlayerPrefsにデータSave/Load

まずUserデータを作成します。育成ゲームなので、ゲームを終了してもすぐ続きからできるように
あとでUserデータをJsonにして保存する、というのをやるので保存する項目を挙げます。

ユーザー
- 所持金
- キャラクター
　- ユニークID
　- マスターデータID

User.cs (新規)

```C#
using System;
using System.Collections.Generic;
using System.Linq;
using UnityEngine;

[Serializable]
public class User
{
    [SerializeField] private int money;
    [SerializeField] private List<Character> characters;

    public int Money { get { return money; } }

    public List<Character> Characters
    {
        get { return characters ?? (characters = new List<Character>()); }
    }

    public Character NewCharacter(MstCharacter data)
    {
        var uniqueId = (Characters.Count == 0) ? 1 : characters[characters.Count - 1].UniqueID + 1;
        var chara = new Character(uniqueId, data);
        characters.Add(chara);
        money -= data.InitialCost;
        return chara;
    }
}
```

※??はnullかどうかをチェックします。もし、nullだった場合はカッコ内の処理が行われます。

で、次にGameManagerにUserデータをもたせてSave/Loadの部分を書きます。

GameManager.cs (編集)

```C#
using UnityEngine;
using System.Collections;

public class GameManager
: SingletonMonoBehaviour<GameManager>
{
	// 3行追記
	[SerializeField] private User userData = new User();
    public User User { get { return userData; } }
    private const string SaveKey = "SaveData";
	
    private void Start()
    {
    	 // データのロード部分追記
    	 if (PlayerPrefs.HasKey(SaveKey))
        {
            userData = JsonUtility.FromJson<User>(PlayerPrefs.GetString(SaveKey));
        }
        
        MasterDataManager.instance.LoadData(() => 
        {
            PortrateUIManager.instance.Setup();
        });
    }
    
    // Save関数追記
    public void Save()
    {
        PlayerPrefs.SetString(SaveKey, JsonUtility.ToJson(userData));
    }
    
    public static void Log (object log)
    {
        if (Debug.isDebugBuild)
        {
            Debug.Log(log);
        }
    }
}
```	

一応これでデータのSave/Loadができたんだけど、
まだCharacterを購入する部分ができてないのでデータほとんど空っぽです。
どんな感じでJsonが保存されるのかみてみたい場合は
GameManagerのStart関数の中にでもLogを仕込んでみると良いでしょう。

```
GameManager.Log( JsonUtility.ToJson(userData) );
```

### JsonUtilityについて

JsonUtilityはobjectを引数に放り込むだけで勝手にJsonに固めてくれます。
こんなかんじに。

```
JsonUtility.ToJson( object )
```

object -> json
これをシリアライズといい
json -> object
こっちをデシリアライズと呼びますが、
シリアライズでjsonに含めてくれる型は知っておいた方が良いので挙げます。

___シリアライズ可___
・クラス(Serializable)
・構造体(Serializable)
・List<T>
・T[](Array / 配列)
・列挙型

___シリアライズ不可___
・object
・Dictionary
・T[][] (Jagged Array / 多次元配列)
・static なフィールド
・プロパティ

ここからパクってます。以上です。
http://neareal.com/2549/


## ゲームロジック

### キャラクター購入ロジック

キャラクターの購入の流れ確認

```
MentorPurchaseCellのPurchaseButtonを押す
↓
押したときに購入可能かどうかを判定
↓
購入可能であれば所持金をコスト分減らしてMentorTrainingViewにCellを追加する。
```

MentorPurchaseCell.cs (編集)

```C#
using UnityEngine;
using UnityEngine.UI;

public class MentorPurchaseCell : MonoBehaviour
{
    [SerializeField] private Image iconImage;
    [SerializeField] private Text
        nameLabel,
        rarityLabel,
        flavorTextLabel,
        productivityLabel,
        costLabel;

    [SerializeField] private Button purchaseButton;
    
    // ここ追記 -> ※InspectorでPurchaseButtonにCanvasGroupっていうコンポーネントくっつけてね！！
    [SerializeField] private CanvasGroup buttonGroup;

    private bool isSold = false;
    private MstCharacter characterData;

    public void SetValue(MstCharacter data)
    {
        iconImage.sprite = Resources.Load<Sprite>("Face/" + data.ImageId);
        characterData = data;
        nameLabel.text = data.Name;
        rarityLabel.text = "";
        for (int i = 0; i < data.Rarity; i++) { rarityLabel.text += "★"; }
        flavorTextLabel.text = data.FlavorText;
        productivityLabel.text = "生産性(lv.1) : " + data.LowerEnergy;
        costLabel.text = string.Format("¥{0:#,0}", data.InitialCost);
        
        // ここ追記
        var user = GameManager.instance.User;
        var ch = user.Characters.Find(c => c.MasterID == data.ID);
        isSold = (ch == null) ? false : true;
        if (isSold) SoldView();
        if (!characterData.PurchaseAvailable(user.Money)) buttonGroup.alpha = 0.5f;
        purchaseButton.onClick.AddListener(() =>
        {
            if (isSold) return;
            if (!characterData.PurchaseAvailable(user.Money)) return;
            isSold = true;
            SoldView();
            var chara = user.NewCharacter(characterData);
            PortrateUIManager.instance.MentorTrainingView.AddCharacter(chara);
        });
    }
    
    // ここ追記
    private void SoldView()
    {
        buttonGroup.alpha = 0.5f;
        costLabel.text = "sold out";
    }
}
```

これでPurchaseButtonをタップしたら購入できるようになったはず。
まぁまだお金が増える部分作ってないから増えないんですけれども。

### 1タップあたりのもらえるお金を算出するロジック

Character.cs (編集)

```C#
using UnityEngine;

[System.SerializableAttribute]
public class Character 
{
	[SerializeField] private int uid, masterId, level;

	public int UniqueID { get { return uid; } }
	public int MasterId { get { return masterId; } }
	public int Level    { get { return level; } }
	
	public MstCharacter Master
	{
		get
		{
			return MasterDataManager.instance.GetCharacterById(masterId);
		}
	}
	
	public int Power 
	{
		get 
		{
			// 編集
			int power = 
				Master.LowerEnergy 
				+ ( 
					(level - 1) 
					* (Master.UpperEnergy - Master.LowerEnergy) 
					/ (Master.MaxLevel - 1) 
				);
			return power;
		}
	}

	public Character (int uniqueId, MstCharacter data)
	{
		uid = uniqueId;
		level = 1; 
		masterId = data.ID;
	}
	
	public bool IsLevelMax 
	{ 
		get
		{  
			return (level >= Master.MaxLevel) ? true : false;
		} 
	}
}
```

生産性の算出方法を線形補間のみ実装してるので、ここgrowthTypeで分けるといいですね。
成長曲線についてはもうちょい下の方に書いてます。


```線形補間
下限 + ((現在のレベル-1) * (上限 - 下限) / (最大レベル - 1))
```

次に、保持してるキャラクター全員分の生産性の合計が1タップあたりの生産性になるので
キャラクターを保持しているUser.csにそのプロパティを作成します。

User.cs (編集)

```C#
using System;
using System.Collections.Generic;
using System.Linq;
using UnityEngine;

[Serializable]
public class User
{
    [SerializeField] private int money;
    [SerializeField] private List<Character> characters;

    public int Money { get { return money; } }

    public List<Character> Characters
    {
        get { return characters ?? (characters = new List<Character>()); }
    }
    
    // 追記部分
    public int ProductivityPerTap
    {
        get 
        { 
            int sum = characters.Sum(c => c.Power);
            return (sum == 0) ? 1 : sum; 
        }
    }
    
    // 追記。moneyに売り上げを加算
    public void AddMoney(int cost)
    {
        money += cost;
    }

    public Character NewCharacter(MstCharacter data)
    {
        var uniqueId = (Characters.Count == 0) ? 1 : characters[characters.Count - 1].UniqueID + 1;
        var chara = new Character(uniqueId, data);
        characters.Add(chara);
        money -= data.InitialCost;
        return chara;
    }
}
```

キャラクターがまだいない場合はcharactersのPowerの合計値が0になるので、
sumが0のときは1タップ1円にしてあげる。

### 成長曲線
レベルアップによる成長のしかたを 早熟/普通/晩成 で分ける。
growthType(1:早熟, 2:線形, 3,晩成)
Character.csの、生産性を返すパラメータ内でswitchで分けよう
計算式はこちらを参考にすると良いかも
https://blogs.yahoo.co.jp/fermiumbay2/41015350.html


### タップしたらお金が増えるようにする。
___ついでにデバッグで便利なデータ削除ボタンもつける。___

PortrateUIManager.cs (編集)

```C#
using System.Linq;
using UnityEngine;
using UnityEngine.UI;

public class PortrateUIManager
: SingletonMonoBehaviour<PortrateUIManager>
{
    [SerializeField] private MentorPurchaseView mentorPurchaseView;
    [SerializeField] private MentorTrainingView mentorTrainingView;
    public MentorPurchaseView MentorPurchaseView { get { return mentorPurchaseView; } }
    public MentorTrainingView MentorTrainingView { get { return mentorTrainingView; } }
    
    // 追記部分
    [SerializeField] private Button workButton, dataClearButton;
    [SerializeField] private Text 
        moneyLabel,
        autoWorkLabel,
        employeesCountLabel,
        productivityLabel;

    [SerializeField] private Const.View 
        currentView = Const.View.Close, 
        lastView = Const.View.Purchase;
    
    // 追記部分    
    private static User User { get { return GameManager.instance.User; } }

    public void Setup ()
    {
        mentorPurchaseView.SetCells();
        mentorTrainingView.SetCells();
        
        openButton.onClick.AddListener(() => {
            openButton.gameObject.SetActive(false);
            ChangeView(lastView);
        });
        
        // 追記部分
        UpdateView();
        
        // 追記部分
        workButton.onClick.AddListener(() =>
        {
            var power = User.Characters.Sum(c => c.Power);
            if (power == 0) power = 1;
            User.AddMoney(power);
            UpdateView();
        });
		
		 // 追記部分
        dataClearButton.onClick.AddListener(() => 
        {
            PlayerPrefs.DeleteAll();
            UnityEngine.SceneManagement.SceneManager.LoadScene("Main");
        });
    }
    
    // 追記部分
    public void UpdateView()
    {
        moneyLabel.text = string.Format("¥{0:#,0}", User.Money);
        employeesCountLabel.text = string.Format("{0:#,0}人", User.Characters.Count);
        productivityLabel.text = string.Format("¥{0:#,0}", User.ProductivityPerTap);
    }

    public void ChangeView (Const.View nextView) 
    {
        if (currentView == nextView) return;
        lastView = currentView;
        currentView = nextView;
        isMoving = true;
        switch (nextView)
        {
            case Const.View.Purchase:
                mentorPurchaseView.gameObject.SetActive(true);
                mentorTrainingView.gameObject.SetActive(false);
                break;
            case Const.View.Training:
                mentorPurchaseView.gameObject.SetActive(false);
                mentorTrainingView.gameObject.SetActive(true);
                break;
            case Const.View.Close:
                openButton.gameObject.SetActive(true);
                break;
        }
    }
    
    [SerializeField] private Button openButton;
    [SerializeField] private Transform 
        mainPanel, 
        openPoint, 
        closePoint;
    private bool isMoving = false;
    private float 
        easing = 8f,
        maxSpeed = 3f,
        stopDistance = 0.001f;

    private void Update () {
        if (!isMoving) return;
        var target = (currentView == Const.View.Close) ? closePoint : openPoint;

        // position
        Vector3 v = Vector3.Lerp(
            mainPanel.position, 
            target.position, 
            Time.deltaTime * easing) - mainPanel.position;
        if (v.magnitude > maxSpeed) v = v.normalized * maxSpeed;
        mainPanel.position += v;
        
        if (isMoving) {
            float distance = Vector3.Distance(mainPanel.position, target.position);
            if (stopDistance > distance) isMoving = false;
        }
    }
}
```

これができたら、透明な全画面に配置するWorkButtonとちっちゃくDataClearButtonを作成して紐づける。
これでお金を貯めてメンターを購入するところまでできます。

### UniRxでのViewの更新
このゲームでは下記の時に起こるイベントが多くあります。というかほとんどの操作がこれにあたります。
・お金の増減

UniRxを使うと、あるパラメータが変化したときにイベントを実行する、というのが簡単になります。
まずはUniRxをダウンロードしてインポートしましょう。
[![https://gyazo.com/3aafa63424d188da0bde36309038960d](https://i.gyazo.com/3aafa63424d188da0bde36309038960d.png)](https://gyazo.com/3aafa63424d188da0bde36309038960d)

次にUserデータのMoneyをUniRxに書き直していきます。

User.cs (編集)

```C#
using System;
using System.Collections.Generic;
using System.Linq;
using UnityEngine;

// 追加
using UniRx;

[Serializable]
public class User
{
    // 編集
    [SerializeField] private IntReactiveProperty money;
    [SerializeField] private List<Character> characters;
	
    // 編集
    public ReadOnlyReactiveProperty<int> Money
    {
        get { return money.ToReadOnlyReactiveProperty(); }
    }

    public List<Character> Characters
    {
        get { return characters ?? (characters = new List<Character>()); }
    }
    
    public int ProductivityPerTap
    {
        get 
        { 
            int sum = characters.Sum(c => c.Power);
            return (sum == 0) ? 1 : sum; 
        }
    }
    
    // moneyに売り上げを加算
    public void AddMoney(int cost)
    {
        money += cost;
    }

    public Character NewCharacter(MstCharacter data)
    {
        var uniqueId = (Characters.Count == 0) ? 1 : characters[characters.Count - 1].UniqueID + 1;
        var chara = new Character(uniqueId, data);
        characters.Add(chara);
        money.Value -= data.InitialCost;
        return chara;
    }
}
```

```
int money
↓
IntReactiveProperty money
```

この変更に伴い、moneyを使っているところでエラーが出ると思うので、
エラーが全部なくなるまで下記の変更をしてください。

```
money 
↓ 
money.Value
```

では次に、お金が増減することで更新する部分を書き換えます。

PortrateUIManager.cs (編集)

```C#
using System.Linq;
using UnityEngine;
using UnityEngine.UI;

// 追記
using UniRx;

public class PortrateUIManager
: SingletonMonoBehaviour<PortrateUIManager>
{
    [SerializeField] private MentorPurchaseView mentorPurchaseView;
    [SerializeField] private MentorTrainingView mentorTrainingView;
    public MentorPurchaseView MentorPurchaseView { get { return mentorPurchaseView; } }
    public MentorTrainingView MentorTrainingView { get { return mentorTrainingView; } }

    [SerializeField] private Button workButton, dataClearButton;
    [SerializeField] private Text 
        moneyLabel,
        autoWorkLabel,
        employeesCountLabel,
        productivityLabel;

    [SerializeField] private Const.View 
        currentView = Const.View.Close, 
        lastView = Const.View.Purchase;
    
        
    private static User User { get { return GameManager.instance.User; } }

    public void Setup ()
    {
        mentorPurchaseView.SetCells();
        mentorTrainingView.SetCells();
        

        openButton.onClick.AddListener(() => {
        openButton.gameObject.SetActive(false);
        ChangeView(lastView);
        });

        UpdateView();
        
        workButton.onClick.AddListener(() =>
        {
            var power = User.Characters.Sum(c => c.Power);
            if (power == 0) power = 1;
            User.AddMoney(power);
            UpdateView();
        });

        dataClearButton.onClick.AddListener(() => 
        {
            PlayerPrefs.DeleteAll();
            UnityEngine.SceneManagement.SceneManager.LoadScene("Main");
        });
        
        // 追記
        User.Money.Subscribe(_ => { UpdateView(); });
    }
    
    public void UpdateView()
    {
        moneyLabel.text = string.Format("¥{0:#,0}", User.Money.Value);
        employeesCountLabel.text = string.Format("{0:#,0}人", User.Characters.Count);
        productivityLabel.text = string.Format("¥{0:#,0}", User.ProductivityPerTap);
    }

    public void ChangeView (Const.View nextView) 
    {
        if (currentView == nextView) return;
        lastView = currentView;
        currentView = nextView;
        isMoving = true;
        switch (nextView)
        {
            case Const.View.Purchase:
                mentorPurchaseView.gameObject.SetActive(true);
                mentorTrainingView.gameObject.SetActive(false);
                break;
            case Const.View.Training:
                mentorPurchaseView.gameObject.SetActive(false);
                mentorTrainingView.gameObject.SetActive(true);
                break;
            case Const.View.Close:
                openButton.gameObject.SetActive(true);
                break;
        }
    }
    
    [SerializeField] private Button openButton;
    [SerializeField] private Transform 
        mainPanel, 
        openPoint, 
        closePoint;
    private bool isMoving = false;
    private float 
        easing = 8f,
        maxSpeed = 3f,
        stopDistance = 0.001f;

    private void Update () {
        if (!isMoving) return;
        var target = (currentView == Const.View.Close) ? closePoint : openPoint;

        // position
        Vector3 v = Vector3.Lerp(
            mainPanel.position, 
            target.position, 
            Time.deltaTime * easing) - mainPanel.position;
        if (v.magnitude > maxSpeed) v = v.normalized * maxSpeed;
        mainPanel.position += v;
        
        if (isMoving) {
            float distance = Vector3.Distance(mainPanel.position, target.position);
            if (stopDistance > distance) isMoving = false;
        }
    }
}
```

あとGameManagerでのデータの保存も、お金の増減があった瞬間に保存してほしいですよね。

GameManager.cs (編集)

```C#
using UnityEngine;
using System.Collections;

//追記
using UniRx;

public class GameManager
: SingletonMonoBehaviour<GameManager>
{
	[SerializeField] private User userData = new User();
    public User User { get { return userData; } }
    private const string SaveKey = "SaveData";
	
    private void Start()
    {
    	 if (PlayerPrefs.HasKey(SaveKey))
        {
            userData = JsonUtility.FromJson<User>(PlayerPrefs.GetString(SaveKey));
        }
        
        MasterDataManager.instance.LoadData(() => 
        {
            PortrateUIManager.instance.Setup();
        });
        
        // 追記
        userData.Money.Subscribe(_ => { Save(); });
    }
    
    public void Save()
    {
        PlayerPrefs.SetString(SaveKey, JsonUtility.ToJson(userData));
    }
    
    public static void Log (object log)
    {
        if (Debug.isDebugBuild)
        {
            Debug.Log(log);
        }
    }
}
```	

それとMentorPurchaseCellの購入ボタン、PurhcaseButtonですが、
現状購入可能かどうかを判別して購入できない状態だと半透明になるようにしています。
こちらも、所持金の増減があったとき毎にボタンが押せるかどうかの判定を入れたいので入れます。

MentorPurchaseCell.cs (編集)

```C#
using UnityEngine;
using UnityEngine.UI;

// 追記
using UniRx;

public class MentorPurchaseCell : MonoBehaviour
{
    [SerializeField] private Image iconImage;
    [SerializeField] private Text
        nameLabel,
        rarityLabel,
        flavorTextLabel,
        productivityLabel,
        costLabel;

    [SerializeField] private Button purchaseButton;
    [SerializeField] private CanvasGroup buttonGroup;

    private bool isSold = false;
    private MstCharacter characterData;

    public void SetValue(MstCharacter data)
    {
        iconImage.sprite = Resources.Load<Sprite>("Face/" + data.ImageId);
        characterData = data;
        nameLabel.text = data.Name;
        rarityLabel.text = "";
        for (int i = 0; i < data.Rarity; i++) { rarityLabel.text += "★"; }
        flavorTextLabel.text = data.FlavorText;
        productivityLabel.text = "生産性(lv.1) : " + data.LowerEnergy;
        costLabel.text = string.Format("¥{0:#,0}", data.InitialCost);
        
        var user = GameManager.instance.User;
        var ch = user.Characters.Find(c => c.MasterId == data.ID);
        isSold = (ch == null) ? false : true;
        if (isSold) SoldView();
        if (!characterData.PurchaseAvailable(user.Money.Value)) buttonGroup.alpha = 0.5f;
        purchaseButton.onClick.AddListener(() =>
        {
            if (isSold) return;
            if (!characterData.PurchaseAvailable(user.Money.Value)) return;
            isSold = true;
            SoldView();
            var chara = user.NewCharacter(characterData);
            PortrateUIManager.instance.MentorTrainingView.AddCharacter(chara);
        });
        
        // 追記
        user.Money.Subscribe(value => {
            if (isSold) return;
            if (value < data.InitialCost) buttonGroup.alpha = 0.5f;
            else buttonGroup.alpha = 1.0f;
        });
    }
    
    private void SoldView()
    {
        buttonGroup.alpha = 0.5f;
        costLabel.text = "sold out";
    }
}
```

こんな感じでMentorTrainingCellにも入れてみてください💜

### LevelUp

まずはCharacter.csにLevelUpという関数を作っておきます。

Character.cs (編集)

```C#
using UnityEngine;

[System.SerializableAttribute]
public class Character 
{
	[SerializeField] private int uid, masterId, level;

	public int UniqueID { get { return uid; } }
	public int MasterId { get { return masterId; } }
	public int Level { get { return level; } }
	
	public MstCharacter Master
	{
		get
		{
			return MasterDataManager.instance.GetCharacterById(masterId);
		}
	}

	public int Power 
	{
		get 
		{
			int power = 
				Master.LowerEnergy 
				+ ( 
					(level - 1) 
					* (Master.UpperEnergy - Master.LowerEnergy) 
					/ (Master.MaxLevel - 1) 
				);
			return power;
		}
	}

	public Character (int uniqueId, MstCharacter data)
	{
		uid = uniqueId;
		level = 1; 
		masterId = data.ID;
	}
	
	
	// 追記部分
	public void LevelUp ()
	{
		level += 1;
	}
}
```

次にMentorTrainingCell.csのLebelUpButtonに、LevelUpを実行するコードを書きたいところなのだが、
今LevelUpのときのコストをとりあえず一律100円にしてるので、MasterDataManager.csで算出することにする。
こちらで決めた仕様は、下記。
・レアリティ毎に最初のコストが変わる
・初期コストから1レベルごとに1.1倍したコストになる

MasterDataManager.cs (編集)

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;
using System.Linq;

public class MasterDataManager 
: SingletonMonoBehaviour<MasterDataManager>
{
    [SerializeField]
	private List<MstCharacter> characterTable = new List<MstCharacter>();
	public List<MstCharacter> CharacterTable { get { return characterTable; } }
	// キャラクターをIDで引っ張れるようにしておく
	public MstCharacter GetCharacterById(int id)
	{
		return characterTable.Find(c => c.ID == id);
	}
	
	// 追記
	public int GetConsumptionMoney (Character data)
	{
		return (int)(data.Master.InitialCost * Mathf.Pow(1.1f, data.Level-1));
	}

	// PublishしてゲットしたURL
	const string csvUrl = "https://docs.google.com/spreadsheets/d/1mYUT577B26EFcw9ifaXWbMMoUHvbkR_NyHYdh3dc94k/pub?gid=605974578&single=true&output=csv";

    // GameManagerから呼んでもらう
	public void LoadData(UnityAction onFinish)
	{
		ConnectionManager.instance.ConnectionAPI(
			csvUrl, 
			(string result) => {
				var csv = CSVReader.SplitCsvGrid(result);
				for (int i=1; i<csv.GetLength(1)-1; i++) 
				{
					var data = new MstCharacter();
					data.SetFromCSV( GetRaw(csv, i) );
					characterTable.Add(data);
				}
				onFinish();
			}
		);
	}

    private string[] GetRaw (string[,] csv, int row) {
        string[] data = new string[ csv.GetLength(0) ];
        for (int i=0; i<csv.GetLength(0); i++) {
            data[i] = csv[i, row];
        }
        return data;
    }
}
```

### Mathf.Pow
シンプルに Pow(x,y) → xのy乗
http://unitygeek.hatenablog.com/entry/2013/01/08/180649

で、次にLevelUpのためのお金を消費する部分をUser.csに作っておく。

User.cs (編集)

```C#
using System;
using System.Collections.Generic;
using System.Linq;
using UnityEngine;
using UniRx;

[Serializable]
public class User
{
    [SerializeField] private IntReactiveProperty money;
    [SerializeField] private List<Character> characters;
	
    public ReadOnlyReactiveProperty<int> Money
    {
        get { return money.ToReadOnlyReactiveProperty(); }
    }

    public List<Character> Characters
    {
        get { return characters ?? (characters = new List<Character>()); }
    }
    
    public int ProductivityPerTap
    {
        get 
        { 
            int sum = characters.Sum(c => c.Power);
            return (sum == 0) ? 1 : sum; 
        }
    }
    
    // moneyに売り上げを加算
    public void AddMoney(int cost)
    {
        money.Value += cost;
    }
    
    // 追記
    // LevelUp時にコストを消費するところ
    public void ConsumptionLevelUpCost(int cost)
    {
        money.Value -= cost;
    }

    public Character NewCharacter(MstCharacter data)
    {
        var uniqueId = (Characters.Count == 0) ? 1 : characters[characters.Count - 1].UniqueID + 1;
        var chara = new Character(uniqueId, data);
        characters.Add(chara);
        money.Value -= data.InitialCost;
        return chara;
    }
}
```

最後にLevelUpButtonでレベルが上がって表示も更新されるように作ればしゅうりょう

MentorTrainingCell.cs (編集)

```C#
using UnityEngine;
using UnityEngine.UI;

// 追記
using UniRx;

public class MentorTrainingCell : MonoBehaviour
{
    [SerializeField] private Image iconImage;

    [SerializeField] private Text
        nameLabel,
        rarityLabel,
        levelLabel,
        productivityLabel,
        costLabel;

    [SerializeField] private Button 
        levelUpButton, 
        discriptionButton, 
        vrButton;
    [SerializeField] private CanvasGroup levelUpButtonGroup;

    private Character characterData;
    
    // 追記
    private User User { get { return GameManager.instance.User; } }

    public void SetValue(Character data)
    {
        var master = data.Master;
        characterData = data;
        iconImage.sprite = Resources.Load<Sprite>("Face/" + master.ImageId);
        nameLabel.text = master.Name;
        rarityLabel.text = "";
        for (var i = 0; i < master.Rarity; i++) { rarityLabel.text += "★"; }
        UpdateValue();
        
        levelUpButton.onClick.AddListener(() =>
        {
            // 追記
            var cost = CulcLevelUpCost();
            if (User.Money.Value < cost) return;
            if (characterData.IsLevelMax) return;
            characterData.LevelUp();
            User.ConsumptionLevelUpCost(cost);
            UpdateValue();
        });

        discriptionButton.onClick.AddListener(() => {
            // Descriptionを表示させるところを作ってないのでまだ
        });

        vrButton.onClick.AddListener(() => {
            // VRViewを作ってないのでまだ
        });
        
        // 所持金のIntReactiveProperty化で対応しておく予定の部分
        if (User.Money.Value < CulcLevelUpCost()) levelUpButtonGroup.alpha = 0.5f;
        User.Money.Subscribe(value => {
            if (characterData.IsLevelMax) return;
            UpdateValue();
        });
    }

    private int CulcLevelUpCost()
    {
    	 // 編集
        return MasterDataManager.instance.GetConsumptionMoney (characterData);
    }

    private void UpdateValue()
    {
        var master = characterData.Master;
        levelLabel.text = "Lv." + characterData.Level;
        productivityLabel.text = string.Format("生産性 : ¥ {0:#,0} /tap", characterData.Power);
        var cost = CulcLevelUpCost();
        costLabel.text = string.Format("¥{0:#,0}", cost);
        
        // ifの条件を編集
        if (User.Money.Value < cost) levelUpButtonGroup.alpha = 0.5f;
        
        else levelUpButtonGroup.alpha = 1.0f;
        if (characterData.IsLevelMax) 
        {
            levelUpButtonGroup.alpha = 0.5f;
            costLabel.text = "Level Max";
        }
    }
}
```



## Avatar

### Mapを作成してNavMeshを作る

まずはAvatarが歩き回る箱庭を作る。
適当にステージ作りましょう。ちなみに僕はこれを使いました。
http://u3d.as/zxv

その前に一旦フォルダを整理したい。
AssetStore活用し始めるとProjectビューが散らかるのでキレイにまとめてください。
ちなみに余談ですが僕はフォルダ整理するのは好きだけど部屋は汚いです。
![file_order.png](https://qiita-image-store.s3.amazonaws.com/0/25374/7c29cbbc-f0ec-7766-1a0c-56a0b5c71c4a.png "file_order.png")

こんな感じにします。
[![https://gyazo.com/c21a67e1bcabe271083c7a96b0bcb069](https://i.gyazo.com/c21a67e1bcabe271083c7a96b0bcb069.png)](https://gyazo.com/c21a67e1bcabe271083c7a96b0bcb069)

NavMeshのBakeまでサクッと。
[![https://gyazo.com/2970b1686305281ad0c143a341c58045](https://i.gyazo.com/2970b1686305281ad0c143a341c58045.png)](https://gyazo.com/2970b1686305281ad0c143a341c58045)


### Avatar本体を作成する。

Avatarに使うアセットはなんでもよい。
ぼくは最初に目に入ったこいつを使ってみることにする。
[![https://gyazo.com/ac250d222fdbb858f0f3ddc8b6e4d534](https://i.gyazo.com/ac250d222fdbb858f0f3ddc8b6e4d534.png)](https://gyazo.com/ac250d222fdbb858f0f3ddc8b6e4d534)

Prefabをこんなかんじにしてみる。
[![https://gyazo.com/145004d4e9005b4b1338a79fb104af98](https://i.gyazo.com/145004d4e9005b4b1338a79fb104af98.png)](https://gyazo.com/145004d4e9005b4b1338a79fb104af98)

作成したAvatarにくっつけるコンポーネントを作成する。

AvatarController.cs (新規)

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.AI;

public class AvatarController : MonoBehaviour {

	[SerializeField] private SpriteRenderer face;
	[SerializeField] private Transform diveCameraPoint, mainCameraPoint;
	public Transform MainCameraPoint { get { return mainCameraPoint; } }

	private Character characterData;
	public Character Character { get{ return characterData; } }
 
	private NavMeshAgent agent;
	private Transform target;

	public void SetValue (Character data)
	{
		characterData = data;
		face.sprite = Resources.Load<Sprite>("Face/" + data.Master.ImageId);
	}

	public Transform VRView ()
	{
		face.gameObject.SetActive(false);
		return diveCameraPoint;
	}

	public void InactiveVR ()
	{
		face.gameObject.SetActive(true);
	}

	private void Start()
	{
		agent = GetComponent<NavMeshAgent>();
		target = AvatarManager.instance.StartPoint;
	}

	private void Update()
	{
		if (target == null) return;
		agent.SetDestination(target.position);

		float distance = Vector3.Distance(transform.position, target.position);
		if (distance < 0.2f) 
		{
			target = AvatarManager.instance.GetTarget();
		}
	}

}
```

AvatarManagerも作っとく↓

AvatarManager.cs (新規)

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class AvatarManager 
: SingletonMonoBehaviour<AvatarManager> 
{
	[SerializeField] private GameObject avatarPrefab;
	private List<AvatarController> avatars = new List<AvatarController>();

	[SerializeField] private Transform spawnPoint, startPoint;
	
	private GameObject[] points;

	public Transform SpawnPoint { get { return spawnPoint; } }
	public Transform StartPoint { get { return startPoint; } }

	public Transform GetTarget()
	{
		int target = Random.Range(0, points.Length-1);
		return points[target].transform;
	}

	private void Start()
	{
		points = GameObject.FindGameObjectsWithTag("WalkPoints");
	}

	public AvatarController GetAvatar (int uniqueId)
	{
		return avatars.Find(a => a.Character.UniqueID == uniqueId);
	}

	public void Setup () { StartCoroutine( SetupCoroutine() ); }
	private IEnumerator SetupCoroutine () 
	{
		var characters = GameManager.instance.User.Characters;
		foreach (var c in characters)
		{
			SpawnAvatar(c);
			yield return new WaitForSeconds(5.0f);
		}
	}

	public void SpawnAvatar(Character data)
	{
		var obj = Instantiate(
			avatarPrefab, 
			spawnPoint.position, 
			spawnPoint.rotation) as GameObject;
		var avatar = obj.GetComponent<AvatarController>();
		avatar.SetValue(data);
		avatars.Add(avatar);
	}

}
```

AvatarControllerをさきほど作成したAvatarにくっつけて紐付け作業をする。
ついでにCharacterControllerとNavMeshAgentもくっつけてPrefab化。
MainCameraPointはキャラクター詳細Popupのときに後ほど。
[![https://gyazo.com/c1d57cc52fb1092cd3fd86d287bbd9e8](https://i.gyazo.com/c1d57cc52fb1092cd3fd86d287bbd9e8.png)](https://gyazo.com/c1d57cc52fb1092cd3fd86d287bbd9e8)

↑選んだAssetによっては向いてる方向とかおかしかったりするので、
サイズ感の調節だけでなく、見た目の調節なんかはPivotの１つ下の子オブジェクトで調節してください。


Avatarが歩き回るようのpointsをからのゲームオブジェクトでいくつか作って
tagを "WalkPoints" という名前に変更しておく。

CreateEmptyで空のGameObjectで作ります。

[![https://gyazo.com/d2ffa03a784bee232c1db4e7e8ea120f](https://i.gyazo.com/d2ffa03a784bee232c1db4e7e8ea120f.png)](https://gyazo.com/d2ffa03a784bee232c1db4e7e8ea120f)


Avatarを生成する座標と最初に歩いて向かう座標用のPointを用意する。
[![https://gyazo.com/272c848cb2b1fac6b14cce74317fa46a](https://i.gyazo.com/272c848cb2b1fac6b14cce74317fa46a.png)](https://gyazo.com/272c848cb2b1fac6b14cce74317fa46a)

※この2つのPointはうろうろ歩き回る用のPointに混ぜたくないのでUntaggedにしておく。
[![https://gyazo.com/7a98fbf8c52172754ec984bbe5c727e4](https://i.gyazo.com/7a98fbf8c52172754ec984bbe5c727e4.png)](https://gyazo.com/7a98fbf8c52172754ec984bbe5c727e4)

AvatarManagerにPrefabとSpawnPoint、StartPointを紐付ける。
ほんとうはInspectorで自分の管理下以外のものを紐付けるのは僕のポリシーに反するので
StageControllerみたいなやつにStartPointを返すプロパティとかで返させたい。
[![https://gyazo.com/91ae041567656fb698a3459d8d008218](https://i.gyazo.com/91ae041567656fb698a3459d8d008218.png)](https://gyazo.com/91ae041567656fb698a3459d8d008218)


AvatarManagerにAvatarを生成するところを作ったので、
GameManagerとMentorPurchaseCellからそこを呼びます。

GameManager.cs (編集)

```C#
using UnityEngine;
using System.Collections;

public class GameManager
: SingletonMonoBehaviour<GameManager>
{
    private void Start()
    {
        MasterDataManager.instance.LoadData(() => 
        {
            PortrateUIManager.instance.Setup();
            // 追記
            AvatarManager.instance.Setup();
        });
    }
    
    public static void Log (object log)
    {
        if (Debug.isDebugBuild)
        {
            Debug.Log(log);
        }
    }
}
```	

MentorPurchaseCell.cs (編集)

```C#
using UnityEngine;
using UnityEngine.UI;
using UniRx;

public class MentorPurchaseCell : MonoBehaviour
{
    [SerializeField] private Image iconImage;
    [SerializeField] private Text
        nameLabel,
        rarityLabel,
        flavorTextLabel,
        productivityLabel,
        costLabel;

    [SerializeField] private Button purchaseButton;
    [SerializeField] private CanvasGroup buttonGroup;

    private bool isSold = false;
    private MstCharacter characterData;

    public void SetValue(MstCharacter data)
    {
        iconImage.sprite = Resources.Load<Sprite>("Face/" + data.ImageId);
        characterData = data;
        nameLabel.text = data.Name;
        rarityLabel.text = "";
        for (int i = 0; i < data.Rarity; i++) { rarityLabel.text += "★"; }
        flavorTextLabel.text = data.FlavorText;
        productivityLabel.text = "生産性(lv.1) : " + data.LowerEnergy;
        costLabel.text = string.Format("¥{0:#,0}", data.InitialCost);
        
        var user = GameManager.instance.User;
        var ch = user.Characters.Find(c => c.MasterId == data.ID);
        isSold = (ch == null) ? false : true;
        if (isSold) SoldView();
        if (!characterData.PurchaseAvailable(user.Money.Value)) buttonGroup.alpha = 0.5f;
        purchaseButton.onClick.AddListener(() =>
        {
            if (isSold) return;
            if (!characterData.PurchaseAvailable(user.Money.Value)) return;
            isSold = true;
            SoldView();
            var chara = user.NewCharacter(characterData);
            PortrateUIManager.instance.MentorTrainingView.AddCharacter(chara);
            
            // 追記
            AvatarManager.instance.SpawnAvatar(chara);
        });
        
        user.Money.Subscribe(value => {
            if (isSold) return;
            if (value < data.InitialCost) buttonGroup.alpha = 0.5f;
            else buttonGroup.alpha = 1.0f;
        });
    }
    
    private void SoldView()
    {
        buttonGroup.alpha = 0.5f;
        costLabel.text = "sold out";
    }
}
```


## VR View (Avaterの視点と切り替えられるようにする。)

やること
・スマホにビルドした時に縦画面と横画面を切り替えられるようにする
・カメラをVR用のカメラに切り替えるようにする

まずGoogleCardboardをインポートする
こちらからGVR Unity v1.40.0をダウンロード
（最新版のv1.50.0はGvrViewerMainというPrefabがないのでv1.40.0で＞＜）↓
https://github.com/googlevr/gvr-unity-sdk/releases

GoogleVRForUnity.unitypackage を開いてインポートする。
[![https://gyazo.com/6b8248823f6841765a4ec8000e868e40](https://i.gyazo.com/6b8248823f6841765a4ec8000e868e40.png)](https://gyazo.com/6b8248823f6841765a4ec8000e868e40)

GoogleVR/Prefabs/GvrViewMain をシーン上に入れてPlayすれば勝手に最初においてあるカメラがVRなカメラになる。
[![https://gyazo.com/1653b0b1fbb2b48ef6a3db9a50c85e79](https://i.gyazo.com/1653b0b1fbb2b48ef6a3db9a50c85e79.png)](https://gyazo.com/1653b0b1fbb2b48ef6a3db9a50c85e79)

が、このアプリでは勝手に初めからVRになられても困るので
Portrate(縦画面)と横画面のVRViewの切り替えをハンドリングするスクリプトを作成する。


DeviceManager.cs (新規)

```C#
using UnityEngine;
using System.Collections;

public class DeviceManager 
: SingletonMonoBehaviour<DeviceManager>
{
	[SerializeField] 
	private GameObject mainCamera, diveCamera, gvrViewer;

	private AvatarController avatarController;

	public void Setup () 
	{ 
		Screen.autorotateToPortrait = false; 
		Screen.autorotateToLandscapeLeft = false; 
		Screen.autorotateToLandscapeRight = false; 
		Screen.autorotateToPortraitUpsideDown = false;
		ToPortrate(); 
	}

	public void ToVR(AvatarController avatar = null) {
		StartCoroutine(ToVRCoroutine(avatar));
	}
	private IEnumerator ToVRCoroutine (AvatarController avatar)
	{
		Screen.orientation = ScreenOrientation.LandscapeLeft;
		avatarController = avatar;
		mainCamera.SetActive(false);
		mainCamera.GetComponent<Camera>().enabled = false;
		diveCamera.SetActive(true);
		if (avatar != null) diveCamera.transform.SetParentWithReset(avatar.VRView());
		yield return null;	// 1フレーム遅らせる
		gvrViewer.SetActive(true);
	}

	public void ToPortrate()
	{
		if (avatarController != null) avatarController.InactiveVR();
		mainCamera.SetActive(true);
		mainCamera.GetComponent<Camera>().enabled = true;
		diveCamera.SetActive(false);
		gvrViewer.SetActive(false);
		Screen.orientation = ScreenOrientation.Portrait;
	}

	private void EnableGvrView ()
	{
		gvrViewer.SetActive(true);
	}

	private void UnenableGvrView ()
	{
		gvrViewer.SetActive(false);
	}
}
```

GameManagerの下にDeviceManagerくっつけて、
DiveCameraは新しくCameraを作成します。
[![https://gyazo.com/16218b8a4c213378cd45fcdf3e2de09a](https://i.gyazo.com/16218b8a4c213378cd45fcdf3e2de09a.png)](https://gyazo.com/16218b8a4c213378cd45fcdf3e2de09a)

ちょっとここSetActiveでGvrViewerをon/offするのダサいのであとで
GvrViewer.VRModeEnabledをいじる方向に修正したい。

で、VRViewからPortrateに戻るボタンがほしいので作ります。
まずScript作成します。

VRUIManager.cs (新規)

```C#
using UnityEngine;
using UnityEngine.UI;

public class VRUIManager : MonoBehaviour {

	[SerializeField]
	private Button toPortrateButton;
	
	void Start () {
		toPortrateButton.onClick.AddListener(() => {
			DeviceManager.instance.ToPortrate();
		});
	}
	
}
```

[![https://gyazo.com/8b6cc1491ca9a760a46831c1e1caa253](https://i.gyazo.com/8b6cc1491ca9a760a46831c1e1caa253.png)](https://gyazo.com/8b6cc1491ca9a760a46831c1e1caa253)

今回はCanvasの ___RenderMode___ を ___ScreenSpace-Overlay___ のまま使っているので
ToPortrateButtonのアンカーをきっちりあわせてあげたりしてあげないと画面サイズが変わったときにどっかいっちゃいます。
[![https://gyazo.com/924a974560f7c87b87e4b44783879c81](https://i.gyazo.com/924a974560f7c87b87e4b44783879c81.png)](https://gyazo.com/924a974560f7c87b87e4b44783879c81)

最後にMentorTrainingCellのVRButtonの処理の中にToVRを入れてあげればVRModeとの切り替えができるようになる。

MentorTrainingCell.cs (編集)

```C#
using UnityEngine;
using UnityEngine.UI;
using UniRx;

public class MentorTrainingCell : MonoBehaviour
{
    [SerializeField] private Image iconImage;

    [SerializeField] private Text
        nameLabel,
        rarityLabel,
        levelLabel,
        productivityLabel,
        costLabel;

    [SerializeField] private Button 
        levelUpButton, 
        discriptionButton, 
        vrButton;
    [SerializeField] private CanvasGroup levelUpButtonGroup;

    private Character characterData;
    
    private User User { get { return GameManager.instance.User; } }
    
    // 追記
    private AvatarController avatar
    {
        get
        {
            return AvatarManager.instance.GetAvatar(characterData.UniqueID);
        }
    }

    public void SetValue(Character data)
    {
        var master = data.Master;
        characterData = data;
        iconImage.sprite = Resources.Load<Sprite>("Face/" + master.ImageId);
        nameLabel.text = master.Name;
        rarityLabel.text = "";
        for (var i = 0; i < master.Rarity; i++) { rarityLabel.text += "★"; }
        UpdateValue();
        
        levelUpButton.onClick.AddListener(() =>
        {
            var cost = CulcLevelUpCost();
            if (User.Money.Value < cost) return;
            if (characterData.IsLevelMax) return;
            characterData.LevelUp();
            User.ConsumptionLevelUpCost(cost);
            UpdateValue();
        });

        discriptionButton.onClick.AddListener(() => {
            // Descriptionを表示させるところを作ってないのでまだ
        });

        vrButton.onClick.AddListener(() => {
            // 追記
            if (avatar == null) return;
            DeviceManager.instance.ToVR(avatar);
        });
        
        if (User.Money.Value < CulcLevelUpCost()) levelUpButtonGroup.alpha = 0.5f;
        User.Money.Subscribe(value => {
            if (characterData.IsLevelMax) return;
            UpdateValue();
        });
    }

    private int CulcLevelUpCost()
    {
        return MasterDataManager.instance.GetConsumptionMoney (characterData);
    }

    private void UpdateValue()
    {
        var master = characterData.Master;
        levelLabel.text = "Lv." + characterData.Level;
        productivityLabel.text = string.Format("生産性 : ¥ {0:#,0} /tap", characterData.Power);
        var cost = CulcLevelUpCost();
        costLabel.text = string.Format("¥{0:#,0}", cost);
        
        // ifの条件を編集
        if (User.Money.Value < cost) levelUpButtonGroup.alpha = 0.5f;
        
        else levelUpButtonGroup.alpha = 1.0f;
        if (characterData.IsLevelMax) 
        {
            levelUpButtonGroup.alpha = 0.5f;
            costLabel.text = "Level Max";
        }
    }
}
```

## Popup

### 全Popup共通して使う機能を作る

IPopupController.cs (新規)

```C#
using UnityEngine;
using UnityEngine.Events;

public interface IPopupController {

	Transform origin { get; }
	void Open (UnityAction onOpenFinish);
	void Close (UnityAction onCloseFinish);

}
```

BasePopupController.cs (新規)

```C#
using UnityEngine;
using UnityEngine.Events;
using System.Collections;

public class BasePopupController : MonoBehaviour, IPopupController {

	private Transform m_Origin;
	public Transform origin { get { return m_Origin; } }

	protected virtual void OnOpenBeforeActive() {}
	protected virtual void OnOpenAfterActive() {}
	protected virtual void OnCloseBeforeActive() {}
	protected virtual void OnCloseAfterActive() {}

	private Animator anim;
	private UnityEngine.Events.UnityAction m_OnOpenFinish;
	private UnityEngine.Events.UnityAction m_OnCloseFinish;

	private void Setup () {
		m_Origin = transform.Find("Popup");
		anim = GetComponent<Animator>();
		transform.localScale = Vector3.one;
	}

	public void Open (UnityAction onOpenFinish = null) {
		Setup();
		m_OnOpenFinish = onOpenFinish;
		this.OnOpenBeforeActive();
	}

	public void Close (UnityAction onCloseFinish = null) {
		m_OnCloseFinish = onCloseFinish;
		this.OnCloseBeforeActive();
		anim.SetTrigger("Close");
	}

	// Animatorから呼ばれる
	private void OnOpenFinish() {
		if (m_OnOpenFinish != null) {
			m_OnOpenFinish();
		}
		this.OnOpenAfterActive();
	}

	// Animatorから呼ばれる
	private void OnCloseFinish() {
		PopupManager.instance.RemoveLastPopup();
		if (m_OnCloseFinish != null) {
			m_OnCloseFinish();
		}
		this.OnCloseAfterActive();
		Destroy(gameObject);
	}
}
```

### 修飾子 protected virtual
※今回は派生クラスで使っていませんが、ぼくがPopupを作るときに使いまわしてるコードなので一旦入れてます。
カリキュラムに入れるかどうかはあとで考えよ

・protected
派生クラスから

・virtual
overrideを使用してオーバーライド可能

ちなみに下記メソッド、

```
protected virtual void OnOpenBeforeActive() {}
protected virtual void OnOpenAfterActive() {}
```

派生クラスでどのように使うかというと

```
protected override void OnCloseAfterActive()
{
    // Popupが開き始めたときにやってほしいこと
}
protected override void OnCloseAfterActive()
{
    // Popupが開き終わったときにやってほしいこと
}
```

こんなかんじ。だが今回は使ってない。

### 汎用的にメッセージを出すPopup
Popup本体をつくります。

CommonPopupController.cs (新規)

```C#
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.Events;

public class CommonPopupController : BasePopupController
{

	[SerializeField] private Text contentLabel;

	[SerializeField] private Button closeButton;

	private UnityAction onCloseFinish;

	private void Start () {
		closeButton.onClick.AddListener(() => {
			if (onCloseFinish != null) Close( onCloseFinish );
			else Close( null );
		});
	}

	public void SetValue (string text, UnityAction OnCloseFinish = null) {
		contentLabel.text = text;
		if (OnCloseFinish != null) onCloseFinish = OnCloseFinish;
	}
}
```

PopupManager.cs (新規)

```C#
using UnityEngine;
using UnityEngine.Events;
using System.Collections;
using System.Collections.Generic;

public class PopupManager
: SingletonMonoBehaviour<PopupManager> {

	public bool isOpened {
		get {
			return popupList.Count > 0 ? true : false;
		}
	}

	// PrefabをInspectorで紐付け
	[SerializeField]
	private GameObject commonPopup;

	private List<IPopupController> popupList = new List<IPopupController>();

	public void RemoveLastPopup () {
		if (popupList.Count > 0) 
			popupList.RemoveAt(popupList.Count-1);
	}

	public void OpenCommon (string message, UnityAction onCloseFinish = null) {
		var popup = CreatePopup( commonPopup );
		var popupController = popup.GetComponent<CommonPopupController>();
		popupController.SetValue( message, onCloseFinish );
		popupList.Add( popupController );
		popupController.Open(null);
	}

	public GameObject CreatePopup (GameObject prefab) {
		GameObject popup = Instantiate( prefab );
		popup.transform.SetParentWithReset( this.transform );
		return popup;
	}
	
}
```

いい感じにUIを組みましょう。
[![https://gyazo.com/1d78cf2d1def43674e3e8df1ca6c3783](https://i.gyazo.com/1d78cf2d1def43674e3e8df1ca6c3783.png)](https://gyazo.com/1d78cf2d1def43674e3e8df1ca6c3783)

作成したCommonPopup(AnimatorController)を開いて、
OpenとCloseを中に入れてしまい、CloseっていうTriggerで Opne -> Close に遷移するようにします。
遷移条件のHasExitTimeのチェックを外すのを忘れずに。
[![https://gyazo.com/459cb5cd3a596c97730652d5fc494252](https://i.gyazo.com/459cb5cd3a596c97730652d5fc494252.png)](https://gyazo.com/459cb5cd3a596c97730652d5fc494252)

Closeが終了したらDestroyするっていうコードをBasePopupControllerに書いているので
Animationの遷移は Open -> Close の一方通行でよいです。

ではCommonPopupにAnimatorをくっつけて、Animationを作ってください。
[![https://gyazo.com/dfc6588519e79ae6441349b412a4d5df](https://i.gyazo.com/dfc6588519e79ae6441349b412a4d5df.png)](https://gyazo.com/dfc6588519e79ae6441349b412a4d5df)

僕が作ったのはこんなAnimationですが、
中心からポコッとでるようなアニメーションを作りたくば、
PopupのScaleを 0 -> 1 になるようなキーを打てばいいと思います。
[![https://gyazo.com/ded76878586702d23bcddef40cc31e65](https://i.gyazo.com/ded76878586702d23bcddef40cc31e65.gif)](https://gyazo.com/ded76878586702d23bcddef40cc31e65)

Animationのキーフレーム(?)秒数が書いてあるところのすぐ下辺りを右クリックすると
Add Animation Event っていう項目が選択できるのでクリック。
[![https://gyazo.com/b45772982f8431fcef74b06a41ad2ea9](https://i.gyazo.com/b45772982f8431fcef74b06a41ad2ea9.png)](https://gyazo.com/b45772982f8431fcef74b06a41ad2ea9)

で、作成したAnimationEventに OnOpenFinish() を登録しておくと、
Popupが開ききったときにOnOpenFinish()が呼ばれてくれる。
[![https://gyazo.com/8bc94272c5e3efd3aa3c679aa22bfa69](https://i.gyazo.com/8bc94272c5e3efd3aa3c679aa22bfa69.png)](https://gyazo.com/8bc94272c5e3efd3aa3c679aa22bfa69)

___CloseのAnimationも作成してOnCloseFinish()も登録したらOK!!!___

できたらPrefab化してHierarchyから削除。
[![https://gyazo.com/958e872090387b11f890a89b61b95c34](https://i.gyazo.com/958e872090387b11f890a89b61b95c34.png)](https://gyazo.com/958e872090387b11f890a89b61b95c34)

PopupManagerにCommonPopupのPrefabを紐付けて準備OK
[![https://gyazo.com/58d1318ee3245176e610d548cee692cf](https://i.gyazo.com/58d1318ee3245176e610d548cee692cf.png)](https://gyazo.com/58d1318ee3245176e610d548cee692cf)


あとはMentorを購入したときにCommonPopupを呼び出すコードを追記する。

MentorPurchaseCell.cs (編集)

```C#
using UnityEngine;
using UnityEngine.UI;

public class MentorPurchaseCell : MonoBehaviour
{
    [SerializeField] private Image iconImage;
    [SerializeField] private Text
        nameLabel,
        rarityLabel,
        flavorTextLabel,
        productivityLabel,
        costLabel;

    [SerializeField] private Button purchaseButton;
    [SerializeField] private CanvasGroup buttonGroup;

    private bool isSold = false;
    private MstCharacter characterData;

    public void SetValue(MstCharacter data)
    {
        iconImage.sprite = Resources.Load<Sprite>("Face/" + data.ImageId);
        characterData = data;
        nameLabel.text = data.Name;
        rarityLabel.text = "";
        for (int i = 0; i < data.Rarity; i++) { rarityLabel.text += "★"; }
        flavorTextLabel.text = data.FlavorText;
        productivityLabel.text = "生産性(lv.1) : " + data.LowerEnergy;
        costLabel.text = string.Format("¥{0:#,0}", data.InitialCost);
        
        var user = GameManager.instance.User;
        var ch = user.Characters.Find(c => c.MasterId == data.ID);
        isSold = (ch == null) ? false : true;
        if (isSold) SoldView();
        if (!characterData.PurchaseAvailable(user.Money.Value)) buttonGroup.alpha = 0.5f;
        purchaseButton.onClick.AddListener(() =>
        {
            if (isSold) return;
            if (!characterData.PurchaseAvailable(user.Money)) return;
            isSold = true;
            SoldView();
            var chara = user.NewCharacter(characterData);
            PortrateUIManager.instance.MentorTrainingView.AddCharacter(chara);
            
            // 追記
            PopupManager.instance.OpenCommon(characterData.Name+"が入社しました！");
        });
        
        user.Money.Subscribe(value => {
            if (isSold) return;
            if (value < data.InitialCost) buttonGroup.alpha = 0.5f;
            else buttonGroup.alpha = 1.0f;
        });
    }
    
    private void SoldView()
    {
        buttonGroup.alpha = 0.5f;
        costLabel.text = "sold out";
    }
}
```

完璧だ！！！完ペキンダックだ！！！！！


### キャラクター詳細Popup

DescriptionPopupController.cs (新規)

```C#
using UnityEngine;
using UnityEngine.UI;
using UnityEngine.Events;
using System.Collections;

public class DescriptionPopupController : BasePopupController
{

	[SerializeField] private Text
        nameLabel,
        flavorTextLabel,
        rarityLabel,
        levelLabel,
        productivityLabel;

	[SerializeField] 
	private Button closeButton;

	private UnityAction onCloseFinish;

	private void Start () 
	{
		closeButton.onClick.AddListener(() => {
			if (onCloseFinish != null) Close( onCloseFinish );
			else Close( null );
		});
	}

	public void SetValue (Character data, UnityAction OnCloseFinish = null) 
	{
        var master = data.Master;
		nameLabel.text = master.Name;
		flavorTextLabel.text = master.FlavorText;
        productivityLabel.text = string.Format("生産性 : ¥ {0:#,0} /tap", data.Power);
        levelLabel.text = "Lv." + data.Level;
        rarityLabel.text = "";
        for (var i = 0; i < master.Rarity; i++) { rarityLabel.text += "★"; }
		if (OnCloseFinish != null) onCloseFinish = OnCloseFinish;
	}
}
```

CommonPopupのときと同じようにUIを組んでみてください。
そのあとPopupManagerへの紐付けもお忘れなく↓

PopupManager.cs (編集)

```C#
using UnityEngine;
using UnityEngine.Events;
using System.Collections;
using System.Collections.Generic;

public class PopupManager
: SingletonMonoBehaviour<PopupManager> {

	public bool isOpened {
		get {
			return popupList.Count > 0 ? true : false;
		}
	}

	// Prefabを手動で紐付け
	[SerializeField]
	private GameObject 
		commonPopup,
		descriptionPopup; // 追記

	private List<IPopupController> popupList = new List<IPopupController>();

	public void RemoveLastPopup () {
		if (popupList.Count > 0) 
			popupList.RemoveAt(popupList.Count-1);
	}

	public void OpenCommon (string message, UnityAction onCloseFinish = null) {
		var popup = CreatePopup( commonPopup );
		var popupController = popup.GetComponent<CommonPopupController>();
		popupController.SetValue( message, onCloseFinish );
		popupList.Add( popupController );
		popupController.Open(null);
	}

	// 追記
	public void OpenDescription (Character data, UnityAction onCloseFinish = null) {
		var popup = CreatePopup( descriptionPopup );
		var popupController = popup.GetComponent<DescriptionPopupController>();
		popupController.SetValue( data, onCloseFinish );
		popupList.Add( popupController );
		popupController.Open(null);
	}

	public GameObject CreatePopup (GameObject prefab) {
		GameObject popup = Instantiate( prefab );
		popup.transform.SetParentWithReset( this.transform );
		return popup;
	}
}
```

キャラクターの詳細時、MainCameraをキャラクターに寄せたいのでカメラ動かすスクリプトを作成する。

MainCameraController.cs (新規)

```C#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.Events;

public class MainCameraController 
: SingletonMonoBehaviour<MainCameraController> 
{

	[SerializeField] private Transform mainCamera;

	private float 
		easing = 6f,
		maxSpeed = 2f,
		stopDistance = 0.2f;

	private UnityAction onZoomInFinish;

	private AvatarController targetAvatar;
	private Transform target;
	public void ToZoomIn (AvatarController nextTarget, UnityAction OnZoomInFinish) {
		targetAvatar = nextTarget;
		target = targetAvatar.MainCameraPoint;
		onZoomInFinish = OnZoomInFinish;
	}

	public void ToZoomOut () {
		onZoomInFinish = null;
		target = this.transform;
	}

	private void Start ()
	{
		target = this.transform;
	}

	private void Update ()
	{
		// position
		Vector3 v = Vector3.Lerp(
			mainCamera.position, 
			target.position, 
			Time.deltaTime * easing) - mainCamera.position;
			
		// rotation
		mainCamera.rotation = Quaternion.Lerp(
			mainCamera.rotation,
			target.rotation,
			Time.deltaTime * easing);

		if (v.magnitude > maxSpeed) v = v.normalized * maxSpeed;
		mainCamera.position += v;

		float distance = Vector3.Distance(mainCamera.position, target.position);
		if (stopDistance > distance) 
		{
			if (onZoomInFinish != null) 
			{
				onZoomInFinish();
				onZoomInFinish = null;
			}
		}
	}
	
}
```

ここで、MainCamera周りのヒエラルキー階層をちょっといじる。
MainCameraを右クリックしてCreateEmptyした後、それをMainCameraと親子関係をひっくり返すだけ。
[![https://gyazo.com/a2cc84481a4915e3ffe08b9d10a6f1e9](https://i.gyazo.com/a2cc84481a4915e3ffe08b9d10a6f1e9.png)](https://gyazo.com/a2cc84481a4915e3ffe08b9d10a6f1e9)

これをやる意図は、MainCameraControllerをPivotとして動かしたいため。

あとAvatarのPrefabにMainCameraPointをまだ作ってないはずなのでそれを作る。
[![https://gyazo.com/f608d7445d79477d04e6799ace79f4b6](https://i.gyazo.com/f608d7445d79477d04e6799ace79f4b6.png)](https://gyazo.com/f608d7445d79477d04e6799ace79f4b6)

[![https://gyazo.com/c8b93ca61b8b41e4b82d87d1326701a7](https://i.gyazo.com/c8b93ca61b8b41e4b82d87d1326701a7.png)](https://gyazo.com/c8b93ca61b8b41e4b82d87d1326701a7)

[![https://gyazo.com/3206795b380a9365df886c4136d9c7f9](https://i.gyazo.com/3206795b380a9365df886c4136d9c7f9.png)](https://gyazo.com/3206795b380a9365df886c4136d9c7f9)

あとはMentorTrainingCellのdescriptionButtonに
DescriptionPopupとカメラのズームする部分を追記。

MentorTrainingCell.cs (編集)

```C#
using UnityEngine;
using UnityEngine.UI;
using UniRx;

public class MentorTrainingCell : MonoBehaviour
{
    [SerializeField] private Image iconImage;

    [SerializeField] private Text
        nameLabel,
        rarityLabel,
        levelLabel,
        productivityLabel,
        costLabel;

    [SerializeField] private Button 
        levelUpButton, 
        discriptionButton, 
        vrButton;
    [SerializeField] private CanvasGroup levelUpButtonGroup;

    private Character characterData;
    private static User User { get { return GameManager.instance.User; } }

    private AvatarController avatar
    {
        get
        {
            return AvatarManager.instance.GetAvatar(characterData.UniqueID);
        }
    }

    public void SetValue(Character data)
    {
        var master = data.Master;
        characterData = data;
        iconImage.sprite = Resources.Load<Sprite>("Face/" + master.ImageId);
        nameLabel.text = master.Name;
        rarityLabel.text = "";
        for (var i = 0; i < master.Rarity; i++)
        {
            rarityLabel.text += "★";
        }
        UpdateValue();
        
        levelUpButton.onClick.AddListener(() =>
        {
            var cost = CulcLevelUpCost();
            if (User.Money.Value < cost) return;
            if (characterData.IsLevelMax) return;
            characterData.LevelUp();
            User.ConsumptionLevelUpCost(cost);
            UpdateValue();
        });
        
        // 追記
        discriptionButton.onClick.AddListener(() => {
            if (avatar == null) return;
            MainCameraController.instance.ToZoomIn(avatar, () => { 
                PopupManager.instance.OpenDescription(characterData, () => {
                    MainCameraController.instance.ToZoomOut();
                });
            });
        });

        vrButton.onClick.AddListener(() => {
            if (avatar == null) return;
            DeviceManager.instance.ToVR(avatar);
        });

        if (User.Money.Value < CulcLevelUpCost()) levelUpButtonGroup.alpha = 0.5f;
        User.Money.Subscribe(value => {
            if (characterData.IsLevelMax) return;
            UpdateValue();
        });
    }

    private int CulcLevelUpCost()
    {
        return MasterDataManager.instance.GetConsumptionMoney (characterData);
    }

    private void UpdateValue()
    {
        var master = characterData.Master;
        levelLabel.text = "Lv." + characterData.Level;
        productivityLabel.text = string.Format("生産性 : ¥ {0:#,0} /tap", characterData.Power);
        var cost = CulcLevelUpCost();
        costLabel.text = string.Format("¥{0:#,0}", cost);
        if (User.Money.Value < cost) levelUpButtonGroup.alpha = 0.5f;
        else levelUpButtonGroup.alpha = 1.0f;
        if (characterData.IsLevelMax) 
        {
            levelUpButtonGroup.alpha = 0.5f;
            costLabel.text = "Level Max";
        }
    }
}
```

おしまい。

あ、あとあれだ、タップしなくても勝手にお金が溜まっていくっていう機能いるよね。

## サウンド類の管理 (おまけ

後回し
