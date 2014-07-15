``` sql
CREATE TABLE students
(
id INTEGER NOT NULL GENERATED ALWAYS AS IDENTITY (START WITH 1, INCREMENT BY 1),
name VARCHAR(24) NOT NULL,
address VARCHAR(1024),
CONSTRAINT primary_key PRIMARY KEY (id)
) ;
```

``` clojure
(defproject feed "0.1.0-SNAPSHOT"
  :description "FIXME: write description"
  :url "http://example.com/FIXME"
  :license {:name "Eclipse Public License"
            :url "http://www.eclipse.org/legal/epl-v10.html"}
  :dependencies [[org.clojure/clojure "1.6.0"]
                 [org.apache.derby/derby "10.10.2.0"]
                 [org.clojure/java.jdbc "0.3.4"]])
```

``` clojure

(def db {:subprotocol "derby"
         :subname "clojure_test"
         :create true})

(db-do-commands db (create-table-ddl  :mytable  [:id :int "DEFAULT 0"] :table-spec ""))

(insert! db :mytable nil [2])

(query db ["SELECT * FROM mytable"])

(def columns (into [] (map #(vec [(symbol %) :string]) ["col1" "col2"])))
(println columns)


(db-do-commands db (create-table-ddl "position" :table-spec ""))

(execute! db ["create table position (positionId varchar(1000) not null)"])

```

``` clojure
(ns feed.core)

(defn row-match [pk fk] 
  (fn [pk-row fk-row]
    (if (or (empty? fk-row) (empty? pk-row))
      false
      (= (pk-row pk) (fk-row fk)))))

(defn inc-matched[feeds matched-keys]
  (into {} (map #(let [k (key %) v (val %)]
                   (if (k matched-keys)
                     [k (rest (k feeds))]                      
                     [k (k feeds)] ))
                feeds)))

(defn process-map[position feeds fmap]
  (into {} (map #(let [k (key %)
                        v (first (val %))
                        func (k fmap)]
                    (when (func position v) [k v]))
                feeds)))

(def feed-match-fn {:position (row-match 2 2)
                    :positionInd (row-match 0 0)})

(defn empty-aggregate[f]
  (into {} (map #(if-let [k (key %)] [k []]) f)))

(declare aggregate-feed process-feed-row)

(defn aggregate-feed[f]
  (let [related-feeds (select-keys f [:positionInd])
        position (first (f :position))
        position-feed (rest (f :position))]
    (trampoline process-feed-row position position-feed related-feeds (empty-aggregate f))))

(defn process-feed-row[position position-feed related-feeds aggregate]
  (if (nil? position)
    ([])
    (let [matches (process-map position related-feeds feed-match-fn)
          merged-aggregate (merge-with conj aggregate matches)
          remap (inc-matched related-feeds matches)             
          match-related (not (every? empty? (vals matches)))
          next-position-matched ((:position feed-match-fn) position (first position-feed))]
      (cond
       (true? match-related)
         #(process-feed-row position position-feed remap merged-aggregate)
       (true? next-position-matched)
         #(process-feed-row (first position-feed) (rest position-feed) remap (merge-with conj merged-aggregate {:position position}))
       :else
       (cons (merge-with conj  merged-aggregate {:position position})
             (lazy-seq (aggregate-feed (merge remap {:position [position-feed]}))))
       )
      )
    )
  )



(with-open [rdr (clojure.java.io/reader "resources/position.txt")]
  (println (map split-row (line-seq rdr))))

(defn split-row [row]
  (clojure.string/split row #"\t"))
```

``` clojure
(ns feed.core-test
  (:require [clojure.test :refer :all]
            [feed.core :refer :all]))

(deftest row-match-test
  (is (= true ((row-match 0 0) [1,2,2] [1,3,4])))
  (is (= false ((row-match 0 0) [1,2,2] [2,2,2])))
  (is (= false ((row-match 0 0) [1,2,2] [])))
  (is (= false ((row-match 0 0) [] [])))
  (is (= false ((row-match 0 0) [] [1,2,2])))
  (is (= false ((row-match 0 0) nil nil)))  
  )

(deftest inc-matched-test
  (is (= {:position [] :positionInd [[1,2,3][2,3,4]]}
         (inc-matched {:position [[1,2,3]] :positionInd [[1,2,3][2,3,4]]} {:position [[1,2,3]]}))))

(deftest process-map-test
  (is (= {:positionInd [1 2 3] :measure [1 2 3]}
         (process-map [1 2 3]
                      {:measure [[1 2 3][1 3 4]] :positionInd [[1 2 3][1 3 4]]}
                      {:measure (row-match 2 2) :positionInd (row-match 0 0)}))))

(deftest empty-aggregate-test
  (is (= {:position []} (empty-aggregate {:position 1}))))

(deftest aggregate-feed-test
  (is (= {:positionInd [[1 2 3] [1 3 4]] :position [[1 2 3]]}
         (first
          (aggregate-feed {:position [[1 2 3][2 3 4]] :positionInd [[1 2 3][1 3 4]]}))))
  (is (= {:positionInd [[1 2 3] [1 3 4]] :position [[1 2 3] [2 3 3]]}
         (first
          (aggregate-feed {:position [[1 2 3][2 3 3]] :positionInd [[1 2 3][1 3 4]]})))))



(run-tests 'feed.core)
```
