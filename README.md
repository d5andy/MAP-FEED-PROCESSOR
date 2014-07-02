MAP-FEED-PROCESSOR
==================

(ns message.core)

(defn match-line [pk fk] 
  (let [pk-key pk fk-key fk]
    (fn [pk-row fk-row]
      (= (nth pk-row pk-key) (nth fk-row fk-key)))))
(println ((match-line 0 0) '(1,2,2) '(2,2,2)))

(defn re-map-map[x y]
  (into {} (map #(let [k (key %) v (val %)]
                   (if (get y k)
                     [k (rest (k x))]                      
                     [k (k x)] ))
                x)))
(println (re-map-map '{:position [1,2,3] :positionInd [[1,2,3][2,3,4]]} '{:position [1,2,3]}))

(defn process-map[p x fmap]
  (into {} (map #(let [k (key %) v (first (val %)) func (get fmap k)]
                     (if (func p v)
                      [k v]                      
                      [k nil]))
                x)))
(println (process-map '(1 2 3) {:position [[1 2 3][2 3 4]] :positionInd [[1 2 3][1 3 4]]} {:position: (match-line 3 3) :positionInd (match-line 0 0)}))
