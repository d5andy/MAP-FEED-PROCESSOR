MAP-FEED-PROCESSOR
==================

``` clojure
(ns message.core)

(defn match-line [pk fk] 
  (fn [pk-row fk-row]
    (if-let [fk-col (if (empty? fk-row) nil (fk-row fk))]
      (= (pk-row pk) fk-col)
      false
      )))
(println ((match-line 0 0) [1,2,2] [1,2,2]))
(println ((match-line 0 0) [] [1,2,2]))
(println ((match-line 0 0) [1,2,2] []))
(println ((match-line 0 0) [] []))

(defn re-map-map[x y]
  (into {} (map #(let [k (key %) v (val %)]
                   (if (k y)
                     [k (rest (k x))]                      
                     [k (k x)] ))
                x)))
(println (re-map-map '{:position [1,2,3] :positionInd [[1,2,3][2,3,4]]} '{:position [1,2,3]}))

(defn process-map[position feeds fmap]
  (into {} (map #(let [k (key %) v (first (val %)) func (k fmap)]
                     (if (func position v)
                      [k v]                      
                      [k nil]))
                feeds)))
(println (process-map [1 2 3] {:position [[1 2 3][1 3 4]] :positionInd [[1 2 3][1 3 4]]} {:position (match-line 2 2) :positionInd (match-line 0 0)}))

(defn aggregate-feed[f]
  (let [related-feed-match-fn {:position (match-line 2 2) :positionInd (match-line 0 0)}
        pos-match-fn (match-line 2 2)  related-feeds (select-keys f [:positionInd])  position-feed (f :position)]
    (loop [position (first (:position f)) pfeed (rest position-feed) feeds related-feeds aggregate {}]
      (if (nil? position)
        (println aggregate)
        (let [matches (process-map position feeds related-feed-match-fn)
              remap (re-map-map feeds matches)             
              match-related (not (every? empty? (vals (dissoc matches :position))))
              next-position-matched (pos-match-fn position (first pfeed) )
              next-position (false? (or match-related next-position-matched))]
          (do
;;            (println (if (next-position) aggregate "still builing")) 
            (recur (if (match-related) position (first (pfeed :position)))
                   (if (match-related) pfeed (rest pfeed))
                   remap
                   (if (next-position) {} (merge matches aggregate) ))))))))

(defn process-feed-row[position position-feed related-feeds aggregate]
  (if (nil? position)
    (println aggregate)
    (let [matches (process-map position feeds related-feed-match-fn)
          remap (re-map-map feeds matches)             
          match-related (not (every? empty? (vals (dissoc matches :position))))
          next-position-matched (pos-match-fn position (first pfeed) )
          next-position (false? (or match-related next-position-matched))]
      (cond
       ()
       :else (println "die")
       
       )
      
   
   ))


(println (aggregate-feed {:position [[1 2 3][1 3 4]] :positionInd [[1 2 3][1 3 4]]}))

(def feed {:position [[1 2 3]]] :positionInd [[1 2 3][1 3 4]]})
(def feed2 {:position [[1 3 4]] :positionInd [[1 2 3][1 3 4]]})
(println (assoc  feed  :position (rest (:position feed)))

         (println (merge feed feed {:market [1 2 3 ]}))

         (println (true? (and false true)))

(map #{if [:position] [(key %)(rest (val %))]} position [[1 2 3][1 3 4]] :positionInd [[1 2 3][1 3 4]]}
```
