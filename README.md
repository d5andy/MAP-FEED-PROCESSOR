(ns message.core)
(use 'clojure.test)

(defn row-match [pk fk] 
  (fn [pk-row fk-row]
    (if (or (empty? fk-row) (empty? pk-row))
      false
      (= (pk-row pk) (fk-row fk)))))

(deftest row-match-test
  (is (= true ((row-match 0 0) [1,2,2] [1,3,4])))
  (is (= false ((row-match 0 0) [1,2,2] [2,2,2])))
  (is (= false ((row-match 0 0) [1,2,2] [])))
  (is (= false ((row-match 0 0) [] [])))
  (is (= false ((row-match 0 0) [] [1,2,2])))
  (is (= false ((row-match 0 0) nil nil)))  
)

(defn inc-matched[feeds matched-keys]
  (into {} (map #(let [k (key %) v (val %)]
                   (if (k y)
                     [k (rest (k x))]                      
                     [k (k x)] ))
                feeds)))
(deftest inc-matched-test
  (is (= {:position [] :positionInd [[1,2,3][2,3,4]]}
         (inc-matched {:position [[1,2,3]] :positionInd [[1,2,3][2,3,4]]} {:position [[1,2,3]]}))))

(defn process-map[position feeds fmap]
  (into {} (map #(let [k (key %)
                        v (first (val %))
                        func (k fmap)]
                    (when (func position v) [k v]))
                feeds)))
(deftest process-map-test
  (is (= {:positionInd [1 2 3] :measure [1 2 3]}
         (process-map [1 2 3]
                      {:measure [[1 2 3][1 3 4]] :positionInd [[1 2 3][1 3 4]]}
                      {:measure (row-match 2 2) :positionInd (row-match 0 0)}))))

(def feed-match-fn {:position (row-match 2 2) :positionInd (row-match 0 0)})

(defn empty-aggregate[f]
  (into {} (map #(if-let [k (key %)] [k []]) f)))

(deftest empty-aggregate-test
  (is (= {:position []} (empty-aggregate {:position 1}))))

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
          remap (re-map-map related-feeds matches)             
          match-related (not (every? empty? (vals matches)))
          next-position-matched ((:position feed-match-fn) position (first position-feed))]
      (cond
       (true? match-related)
         #(process-feed-row position position-feed remap merged-aggregate)
       (true? next-position-matched)
         #(process-feed-row (first position-feed) (rest position-feed) remap (merge-with conj merged-aggregate {:position position}))
       :else
         (cons (merge-with conj  merged-aggregate {:position position})  (lazy-seq (aggregate-feed (merge remap {:position [position-feed]}))))))))

(deftest aggregate-feed-test
  (is (= {:positionInd [[1 2 3] [1 3 4]] :position [[1 2 3]]}
         (first
          (aggregate-feed {:position [[1 2 3][2 3 4]] :positionInd [[1 2 3][1 3 4]]}))))
  (is (= {:positionInd [[1 2 3] [1 3 4]] :position [[1 2 3] [2 3 3]]}
         (first
          (aggregate-feed {:position [[1 2 3][2 3 3]] :positionInd [[1 2 3][1 3 4]]})))))


(run-tests 'message.core)
```
