# PostgreSQL接続のサンプル

```java
/*
 DB接続のサンプル例
 # データベース一覧を表示
 $ psql -l
 $ # sample_dbを作成する
 $ createdb sample_db
 $ # appuser という データベース利用者を作成する
 $ psql -d sample_db -c "CREATE USER appuser WITH PASSWORD 'password';"
 $ # appuser という データベース利用者に sample_db への全権限を付与する
 $ psql -d sample_db -c "GRANT ALL PRIVILEGES ON DATABASE sample_db TO appuser;"
 $ psql -d sample_db -c "GRANT ALL ON SCHEMA public TO appuser;"
 $ # appuser で sample_db に接続する
 $ psql -d sample_db -U appuser
 # sample_db に接続したら、以下のSQLを実行する
 sample_db=> 
   -- usersテーブルを作成する
   CREATE TABLE users (
     id SERIAL PRIMARY KEY,
     name VARCHAR(100) NOT NULL,
     gender VARCHAR(10) NOT NULL,
     birth_date DATE NOT NULL,
     age INTEGER NOT NULL
   );

   -- データを挿入する (6月1日時点での年齢)
   INSERT INTO users (name, gender, birth_date, age) VALUES ('山田 太郎', '男', '2001-05-10', 24);
   INSERT INTO users (name, gender, birth_date, age) VALUES ('鈴木 花子', '女', '2002-08-22', 22);
   INSERT INTO users (name, gender, birth_date, age) VALUES ('田中 次郎', '男', '2003-01-15', 22);
   INSERT INTO users (name, gender, birth_date, age) VALUES ('佐藤 由紀', '女', '2004-03-30', 21);
   INSERT INTO users (name, gender, birth_date, age) VALUES ('小林 健太', '男', '2005-07-19', 19);
   INSERT INTO users (name, gender, birth_date, age) VALUES ('高橋 美香', '女', '2001-10-05', 23);
   INSERT INTO users (name, gender, birth_date, age) VALUES ('中村 翔太', '男', '2002-12-25', 22);
   INSERT INTO users (name, gender, birth_date, age) VALUES ('山本 恵美', '女', '2002-06-14', 22);
   INSERT INTO users (name, gender, birth_date, age) VALUES ('加藤 陽斗', '男', '2004-09-01', 20);
   INSERT INTO users (name, gender, birth_date, age) VALUES ('松本 彩', '女', '2003-02-18', 22); 

   -- 結果を確認する
   SELECT FROM users;

   -- 未だ誕生日を迎えていない、遅生まれの22歳の女性を抽出する
   SELECT FROM users WHERE gender = '女' AND birth_date BETWEEN '2002-04-02' AND '2002-12-31' AND age = 22 ORDER BY id ASC;

   -- 見やすく改行した例
   SELECT id, name, gender, birth_date, age
     FROM users
    WHERE gender = '女'
      AND birth_date BETWEEN '2002-04-02' AND '2002-12-31'
      AND age = 22
    ORDER BY id ASC;

    -- 出力例
     id |   name    | gender | birth_date | age 
    ----+-----------+--------+------------+-----
      2 | 鈴木 花子 | 女     | 2002-08-22 |  22
      8 | 山本 恵美 | 女     | 2002-06-14 |  22
    (2 行)

    https://jdbc.postgresql.org/ より、
    公式 PostgreSQL JDBC Driver をダウンロード、
    以下でコンパイル、実行する。
    javac -cp postgresql-42.7.7.jar SqlSample.java
    java -cp .:postgresql-42.7.7.jar SqlSample
    */

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.Date;

public class SqlSample {
    public static void main(String[] args) {
        // JDBC接続やSQL実行時の例外に対応
        try {
            // PostgreSQLデータベースに接続
            Connection conn = DriverManager.getConnection(
                "jdbc:postgresql://localhost:5432/sample_db",
                "appuser",
                "password"
            );

            // 利用者情報を抽出するSQL文（プレースホルダを使用）
            String sql = """
                -- 利用者情報を抽出するSQL
                SELECT id, name, gender, birth_date, age     -- 取得するカラム
                  FROM users                                 -- 対象テーブル
                 WHERE gender = ?                            -- 性別（バインド値で指定）
                   AND birth_date BETWEEN ? AND ?            -- 誕生日の範囲指定（バインド値で開始日・終了日）
                   AND age = ?                               -- 年齢（バインド値で指定）
                 ORDER BY id ASC                             -- idの昇順で並び替え
                """;

            // SQL文を準備
            PreparedStatement stmt = conn.prepareStatement(sql);
            // 1番目のプレースホルダに性別「女性」をバインド
            stmt.setString(1, "女");
            // 2番目のプレースホルダに誕生日の開始日をバインド
            stmt.setDate(2, Date.valueOf("2002-04-02"));
            // 3番目のプレースホルダに誕生日の終了日をバインド
            stmt.setDate(3, Date.valueOf("2002-12-31"));
            // 4番目のプレースホルダに年齢「22」をバインド
            stmt.setInt(4, 22);

            // 準備したSQL文（プレースホルダ付き）の内容を出力（デバッグ用）
            System.out.println(stmt.toString());
            // 出力例
            // -- 利用者情報を抽出するSQL
            // SELECT id, name, gender, birth_date, age     -- 取得するカラム
            //   FROM users                                 -- 対象テーブル
            //  WHERE gender = ('女')                       -- 性別（バインド値で指定）
            //    AND birth_date BETWEEN ('2002-04-02 +09') AND ('2002-12-31 +09') -- 誕生日の範囲指定（バインド値で開始日・終了日）
            //    AND age = ('22'::int4)                    -- 年齢（バインド値で指定）
            //  ORDER BY id ASC                             -- idの昇順で並び替え

            // SQL文を実行して結果を取得
            ResultSet rs = stmt.executeQuery();

            // 結果を表示
            while (rs.next()) {
                System.out.printf("%d %s %s %s %d\n",
                    rs.getInt("id"),
                    rs.getString("name"),
                    rs.getString("gender"),
                    rs.getDate("birth_date"),
                    rs.getInt("age")
                );
            }
            // 出力例
            // 2 鈴木 花子 女 2002-08-22 22
            // 8 山本 恵美 女 2002-06-14 22

            // リソースのクローズ
            rs.close();
            stmt.close();
            conn.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```