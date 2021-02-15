## Unityで大量のオブジェクトを出すテスト。

動いてるところは[こちら]("https://youtu.be/XeMQd6l-MOE")。（youtubeへのリンクです）

これは何してるかというと、Unity上で弾幕STGっぽくオブジェクトを出しまくるためのテストです。

以前からやろうかなーとは思ってて、割と時間もかかるということでやめてたんですが、
開発再開するにあたって自分のテンション上がるものを組み込むほうが開発が進むだろうという考えで、
実装をしてみたものです。

やったことは以下の通り。

- GameObject上で動的にMeshを作成
- Materialを事前に複数作成、アタッチしておく
- Update関数を実行するタイミングで大量のオブジェクトを描画、その際はパターンごとに上記Materialを割当
- 動的なバッチングが発生してマテリアルごとにバッチ処理が起動
- 結果、大量オブジェクト生成の割に描画負荷が下がる

ちなみにMaterialの定義をまとめられないかも検証したのですが以下の理由で断念。

- MaterialPropertyBlockはMaterialの数値のいち部分変更だが、結局SetPassCallが増える
- Material上のOffset等UV座標を毎回動的に変更すると別マテリアル扱いになってバッチが走らないため、事前にMaterialごとにUV座標を定義し、Material単位で描画コールした。MaterialPropertyBlockを配列で持っておけばバッチ処理が走るかもしれない
- プロジェクト全体でのMaterialのメモリが104KBだったが、利用可能なメモリの総容量考えて複数搭載しても問題なかろ！と考えてメモリ上に乗せる際の工夫は特になし

以下検証したソースコードです。
実験してた関係上上記の動画とは少し違うかもしれませんがご了承ください。

マテリアルはEditor上で指定して、その際Materialはテクスチャを参照するようにしてください。

```csharp
using UnityEngine;
using System.Collections;
using System.Collections.Generic;
public class DynamicMeshUsingDrawMesh : MonoBehaviour {
    [SerializeField]
    Material m_mat;
    [SerializeField]
    Material m_mat2;
    [SerializeField]
    Material m_mat3;
    [SerializeField]
    Material m_mat4;
    Mesh m_mesh;
    public float buf_x=1.0f;
    public float buf_y=1.0f;
    public float offset_x=0.0f;
    public float offset_y=0.0f;
    private int bullet_num=1024;
    private Vector2[] uv_base;
    private Vector2[] uv_result=new Vector2[4];
    private float delta = 0.01666667f;
    private float _time;
    private float _pre_time;
    public class status{
        public float x;
        public float y;
        public float z;
        public bool isAlive;
        public int count;
        public float dx;
        public float dy;
        public status(){
            x=0.0f;
            y=0.0f;
            isAlive=false;
            count=0;
            dx=0.0f;
            dy=0.0f;
        }
    };
    private Vector2 offset;
    public status[] bullet_status;
    private MaterialPropertyBlock _materialPropertyBlock;
    void Start () {
        // 動的なMesh生成
        m_mesh = new Mesh();
        m_mesh.vertices = new Vector3[] {
            new Vector3 (-0.125f, -0.25f),
            new Vector3 (0.125f, -0.25f),
            new Vector3 (0.125f, 0.25f),
            new Vector3 (-0.125f, 0.25f),
        };
        uv_base = new Vector2[]
        {
            new Vector2(0,0),
            new Vector2(0.25f,0),
            new Vector2(0.25f,1.0f),
            new Vector2(0,1.0f),
        };
        m_mesh.triangles = new int[] {
            0, 1, 2,
            0, 2, 3,
        };
        m_mesh.RecalculateNormals();
        m_mesh.RecalculateBounds();
        bullet_status = new status[bullet_num];
        for(int i=0;i&lt;bullet_num; i++){
            bullet_status[i] = new status();
            bullet_status[i].dx = Random.Range(-0.05f, 0.05f);
            bullet_status[i].dy = Random.Range(-0.05f, 0.05f);
            bullet_status[i].isAlive = true;
            bullet_status[i].count=i;
            bullet_status[i].z = i*0.00001f;
        }
        //_materialPropertyBlock = new MaterialPropertyBlock();
        _time=Time.deltaTime;
        _pre_time=Time.deltaTime;
    }
    private void Update()
    {
        bool isMove = false;
        _time += Time.deltaTime;
        if( _time&gt; delta){
            isMove =true;
            _time=0.0f;
            //Debug.Log(_time);
        }
        selfUpdate(isMove);
    }
    public void selfUpdate(bool isMove){
        //_materialPropertyBlock.Clear();
        Material tmp = m_mat;

        for(int i=0; i&lt;bullet_num; i++ ){
            //描画命令
            if( bullet_status[i].isAlive==true){

                //UV座標の更新
                for( int j=0; j&lt;4; j++){
                    {
                        uv_result[j].x = uv_base[j].x;// + (bullet_status[i].count%4*0.25f);
                    }
                    //Y座標は変化なし
                    uv_result[j].y= uv_base[j].y;
                }
                //UV座標をアニメさせる
                m_mesh.uv = uv_result;
                if( isMove== true){
                    bullet_status[i].x += bullet_status[i].dx;
                    bullet_status[i].y += bullet_status[i].dy;
                }
                if( bullet_status[i].count%4 == 0)
                    tmp = m_mat;
                else if( bullet_status[i].count%4 ==1)
                    tmp = m_mat2;
                else if( bullet_status[i].count%4 == 2)
                {
                    tmp = m_mat3;
                }
                else if( bullet_status[i].count%4 == 3){
                    tmp = m_mat4;
                }
                else{}
/*
                offset.x = bullet_status[i].count%4*0.25f;
                offset.y = 0.0f;
                */
                //_materialPropertyBlock.SetTextureOffset("_MainTex", offset);
                //_materialPropertyBlock.SetVector("_MainTex_ST", new Vector4(1.0f, 1.0f, bullet_status[i].count%4*0.25f, 0.0f));
                Graphics.DrawMesh(m_mesh, new Vector3(bullet_status[i].x,
                                                bullet_status[i].y,
                                                bullet_status[i].z),
                                                Quaternion.AngleAxis(20*(float)(bullet_status[i].count%18), new Vector3(0,0,1)), //Quaternion.identity,
                                                tmp, 0
                                                //,null, 0, _materialPropertyBlock, false, false
                                                );
            }
            //カウント
            if( isMove == true){
                bullet_status[i].count++;
                if( bullet_status[i].count&gt;0)
                    bullet_status[i].isAlive=true;
                //画面外処理
                if( bullet_status[i].x &lt;-5.0f || bullet_status[i].x &gt; 5.0){
                    bullet_status[i].x=0.0f;
                    bullet_status[i].y=0.0f;
                    bullet_status[i].count=-30;
                    bullet_status[i].isAlive=false;
                }else if( bullet_status[i].y &lt;-5.0f || bullet_status[i].y &gt; 5.0){
                    bullet_status[i].x=0.0f;
                    bullet_status[i].y=0.0f;
                    bullet_status[i].count=-30;
                    bullet_status[i].isAlive=false;
                }
            }
        }
    }
}
```

参考URL

[【Unity】DrawMesh初歩と動的Mesh生成 - AkiIroブログ (hatenablog.com)](http://akiiro.hatenablog.com/entry/2017/07/05/002213)

 