MAP-FEED-PROCESSOR
==================

``` clojure
(ns message.core)

(defn match-line [pk fk] 
  (fn [pk-row fk-row]
    (= (nth pk-row pk) (nth fk-row fk))))
(println ((match-line 0 0) '(1,2,2) '(2,2,2)))

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
(println (process-map '(1 2 3) {:position [[1 2 3][1 3 4]] :positionInd [[1 2 3][1 3 4]]} {:position (match-line 2 2) :positionInd (match-line 0 0)}))

(defn aggregate-feed[f]
  (loop [position (first (get f :position))
        feeds f
        fmp {:position (match-line 2 2) :positionInd (match-line 0 0)}]
    (if-not position
      nil
      (let [matches (process-map position feeds fmp)
            remap (re-map-map feeds matches)
            matched (+ (map #(val %) matches))]
        (do
          (println matched)
          (recur (first (get feeds :position)) feeds fmp))))))
(println (aggregate-feed {:position [[1 2 3][1 3 4]] :positionInd [[1 2 3][1 3 4]]}))
```
