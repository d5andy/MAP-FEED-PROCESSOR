```lisp
(custom-set-variables
 ;; custom-set-variables was added by Custom.
 ;; If you edit it by hand, you could mess it up, so be careful.
 ;; Your init file should contain only one such instance.
 ;; If there is more than one, they won't work right.
 '(ansi-color-names-vector
   ["#212526" "#ff4b4b" "#b4fa70" "#fce94f" "#729fcf" "#e090d7" "#8cc4ff" "#eeeeec"])
 '(custom-enabled-themes (quote (Darkula))))
(custom-set-faces
 ;; custom-set-faces was added by Custom.
 ;; If you edit it by hand, you could mess it up, so be careful.
 ;; Your init file should contain only one such instance.
 ;; If there is more than one, they won't work right.
 )
(put 'scroll-left 'disabled nil)

(setq url-proxy-services
     '(("no_proxy" . "^\\(localhost\\|10.*\\)")
     ("http" . "localhost:8080")
     ("https" . "localhost:8080")))
(setq url-http-proxy-basic-auth-storage (list (list
    "localhost:8080"(cons "Input your LDAP UID !"
                      (base64-encode-string "sanderdb:Proxy123")))))

(require 'package)
(add-to-list 'package-archives
             '("marmalade" . "http://marmalade-repo.org/packages/"))
(add-to-list 'package-archives
             '("melpa" . "http://melpa.milkbox.net/pacakges/"))
(package-initialize)

(add-hook 'cider-mode-hook 'cider-turn-on-eldoc-mode)
;;(setq nrepl-hide-special-buffers t)
(setq cider-repl-tab-command 'indent-for-tab-command) 
(setq cider-prefer-local-resources t)
(setq cider-repl-pop-to-buffer-on-connect nil)
;;(setq cider-show-error-buffer nil)
(setq cider-show-error-buffer 'only-in-repl)
(setq cider-auto-select-error-buffer nil)
(setq cider-stacktrace-default-filters '(tooling dup))
(setq cider-stacktrace-fill-column 80)
(setq nrepl-buffer-name-separator "-")
(setq nrepl-buffer-name-show-port t)
(setq cider-repl-display-in-current-window t)
(setq cider-repl-print-length 100) ; the default is nil, no limit
(setq cider-prompt-save-file-on-load nil)
(setq cider-repl-result-prefix ";; => ")
(setq cider-interactive-eval-result-prefix ";; => ")
(setq cider-repl-use-clojure-font-lock t)
(setq cider-switch-to-repl-command 'cider-switch-to-current-repl-buffer)
;;(setq cider-known-endpoints '(("host-a" "10.10.10.1" "7888") ("host-b" "7888")))
(setq cider-repl-wrap-history t)
(setq cider-repl-history-size 1000) ; the default is 500
;;(setq cider-repl-history-file "path/to/file")

(require 'icomplete)

(add-hook 'cider-repl-mode-hook 'subword-mode)
(add-hook 'cider-repl-mode-hook 'paredit-mode)
;;(add-hook 'cider-repl-mode-hook 'rainbow-delimiters-mode)

(add-to-list 'load-path "~/.emacs.d/direx-el/")
(require 'direx)
(global-set-key (kbd "C-x C-j") 'direx:jump-to-directory)
```

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
                     [k (seq (rest (k feeds)))]                      
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
    (trampoline process-feed-row {:position position :positionInd []} position-feed related-feeds [])))

(defn process-feed-row[position position-feed related-feeds aggregate]
  (if (empty? (position :position))
    []
    (let [matches (process-map (position :position) related-feeds feed-match-fn)
          merged-aggregate (merge-with conj position matches)
          remap (inc-matched related-feeds matches)             
          match-related (not (every? empty? (vals matches)))
          next-position-matched ((:position feed-match-fn) (:position position) (first position-feed))]
      (cond
       (true? match-related)
       #(process-feed-row merged-aggregate position-feed remap aggregate)
       (true? next-position-matched)
       #(process-feed-row {:position (first position-feed) :positionInd []} (rest position-feed) remap (conj aggregate merged-aggregate))
       :else
       (cons (conj aggregate merged-aggregate) (lazy-seq (aggregate-feed (merge remap {:position [position-feed]}))))  
       )
      )
    )
  )

(doseq [line (aggregate-feed {:position [[1 2 3][2 3 3][3 3 4]] :positionInd [[1 2 3][1 3 4]]})] (prn "line " line))

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
