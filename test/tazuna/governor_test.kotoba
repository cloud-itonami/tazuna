(ns tazuna.governor-test
  "Tests for the tazuna 手綱 Governor + operation catalogue (ADR-2606042100).

  These assert REFUSAL, and they assert it by name. A test that only checks that
  *something* was refused cannot tell a G3 refusal from a typo, so every negative
  case below pins the `:code` the Governor reported and, where the wording is the
  point, a literal from the detail. (This loop shipped exactly that weaker form on
  2026-09-05 — `(is (seq (:detail v)))` stayed green when the specific reason was
  deleted, because the generic fallback is also a non-empty string.)

  The cross-implementation test at the bottom is the one that would have caught
  the defect this file was written for: three implementations of the teleop
  vocabulary agreed, the fourth did not, and nothing compared them."
  (:require [clojure.test :refer [deftest is]]
            [kotoba.lang.text :as str]
            [tazuna.governor :as governor]
            [tazuna.operation :as op]
            [tazuna.methods.teleop-safety :as ts]
            [tazuna.cells.teleop-session.state-machine :as sm]
            #?(:clj [clojure.edn :as edn])))

(def ^:private well-formed
  {:kind "move" :force-class "soft-actuation" :force-auth-ref "forceauth:sbt/1"
   :member-sig "sig:member" :server-sig ""})

;; kinds that must never be representable — the charter's forbidden verbs, plus
;; the empty string and a plain unknown token.
(def ^:private uncatalogued
  ["weaponize" "fire_payload" "detonate" "arm" "strike" "launch" "" "anything-at-all"])

;; ── N1: deny-by-default on the vocabulary ───────────────────────────────────
(deftest an-uncatalogued-command-kind-is-refused
  (doseq [k uncatalogued]
    (let [d (governor/adjudicate (assoc well-formed :kind k))]
      (is (= :refuse (:decision d)) (str "kind " (pr-str k) " must be refused"))
      (is (= :uncatalogued-command (:code d))
          (str "kind " (pr-str k) " must be refused as uncatalogued, got " (:code d)))
      (is (= 14 (:kernel-code d))
          (str "kind " (pr-str k) " must carry the kernel's code 14"))
      ;; the reason must be the catalogue's own reason, not a generic refusal
      (is (str/includes? (:detail d) "not in the teleop catalogue")
          (str "refusal of " (pr-str k) " must name the catalogue: " (:detail d)))
      (is (str/includes? (:detail d) (pr-str k))
          "refusal must name the kind it refused"))))

(deftest an-uncatalogued-kind-is-refused-even-when-asserted-to-be-a-safety-command
  ;; A caller cannot talk its way past the catalogue by claiming a kind is safety.
  (let [d (governor/adjudicate {:kind "emergency_detonate"})]
    (is (= :uncatalogued-command (:code d)))
    (is (false? (op/always-permitted? "emergency_detonate")))))

;; ── reverse direction: the gate must not close too far ──────────────────────
(deftest every-catalogued-kind-is-admitted-when-well-formed
  (doseq [k (op/command-kinds)]
    (let [d (governor/adjudicate (assoc well-formed :kind k))]
      (is (governor/admitted? d)
          (str "catalogued kind " (pr-str k) " must be admitted, got " (pr-str d))))))

(deftest the-catalogue-names-exactly-the-benign-verbs
  (is (= #{"move" "manipulate" "halt" "estop" "handback"} (op/command-kinds)))
  (is (= #{"observational" "soft-actuation" "powered-actuation"} op/force-classes)))

;; ── G3: Transparent Force ───────────────────────────────────────────────────
(deftest actuation-without-force-authorization-is-refused
  (doseq [k ["move" "manipulate"]]
    (let [d (governor/adjudicate (assoc well-formed :kind k :force-auth-ref ""))]
      (is (= :not-force-authorized (:code d))
          (str k " without a force-auth ref must be refused as G3, got " (:code d)))
      (is (= "G3" (:gate d)))
      (is (= 11 (:kernel-code d)))
      (is (str/includes? (:detail d) "force-authorization reference")))))

;; ── N1: force class ─────────────────────────────────────────────────────────
(deftest an-unrepresentable-force-class-cannot-actuate
  (doseq [fc ["weaponizable" "kinetic" "" "force-as-harm"]]
    (let [d (governor/adjudicate (assoc well-formed :force-class fc))]
      (is (= :unrepresentable-force-class (:code d))
          (str "force class " (pr-str fc) " must be refused as N1, got " (:code d)))
      (is (= 10 (:kernel-code d)))
      (is (str/includes? (:detail d) "unrepresentable")))))

;; ── G4: no-server-key ───────────────────────────────────────────────────────
(deftest a-server-signature-is-refused-for-every-kind
  (doseq [k (op/command-kinds)]
    (let [d (governor/adjudicate (assoc well-formed :kind k :server-sig "sig:server"))]
      (is (= :server-signature-refused (:code d))
          (str "a server signature on " (pr-str k) " must be refused, got " (:code d)))
      (is (= 12 (:kernel-code d)))
      (is (str/includes? (:detail d) "no-server-key")))))

(deftest actuation-requires-a-member-signature
  (doseq [k ["move" "manipulate"]]
    (let [d (governor/adjudicate (assoc well-formed :kind k :member-sig ""))]
      (is (= :member-signature-required (:code d)))
      (is (= 13 (:kernel-code d))))))

;; ── safety commands stay ungated ────────────────────────────────────────────
(deftest safety-commands-are-never-gated-or-signed
  (doseq [k ["halt" "estop" "handback"]]
    (let [d (governor/adjudicate {:kind k :force-class "weaponizable"
                                  :force-auth-ref "" :member-sig "" :server-sig ""})]
      (is (governor/admitted? d)
          (str k " must be honoured with no force auth, no signature, any force class"))
      (is (true? (:safety? d)))
      (is (false? (:actuates? d))))))

;; ── the boundary protocol shared with methods/teleop_safety.kotoba ──────────
(deftest the-catalogue-carries-the-kernel-integer-protocol
  (is (= {"halt" 0 "estop" 1 "handback" 2 "move" 3 "manipulate" 4}
         (into {} (map (juxt identity op/kernel-code)) (op/command-kinds)))
      "kernel codes must match methods/teleop_safety.kotoba's stable protocol")
  (is (= #{3 4} (set (map op/kernel-code (filter op/actuates? (op/command-kinds)))))
      "the kernel tests actuation as (or (= command-kind 3) (= command-kind 4))"))

#?(:clj
   (deftest the-kotoba-kernel-still-refuses-out-of-range-kinds
     ;; The kernel is the canonical authority; if its range check is edited the
     ;; catalogue's claim to mirror it stops being true.
     (let [src (slurp (java.io.File. (java.io.File. (System/getProperty "user.dir"))
                                     "methods/teleop_safety.kotoba"))]
       (is (str/includes? src "(or (< command-kind 0) (> command-kind 4)) 14")
           "the kernel must still refuse an out-of-range command kind with code 14")
       (is (str/includes? src "(and (or (= command-kind 3) (= command-kind 4))")
           "the kernel must still treat 3/4 as the actuating kinds"))))

;; ── the catalogue is a projection of the lexicon, not a fifth opinion ───────
#?(:clj
   (deftest the-catalogue-is-the-lexicon-vocabulary
     (let [tx   (edn/read-string
                 (slurp (java.io.File. (java.io.File. (System/getProperty "user.dir"))
                                       "lex/teleopCommand.edn")))
           defs (edn/read-string (:lex/defs (first tx)))
           enum (set (get-in defs [:main :record :properties :kind :enum]))]
       (is (= enum (op/command-kinds))
           (str "lex/teleopCommand.edn kind enum " enum
                " must equal the catalogue " (op/command-kinds))))))

;; ── cross-implementation parity: the test the defect needed ────────────────
(deftest the-cell-refuses-exactly-what-the-reasoner-refuses
  ;; Before 2026-09-06 the reasoner refused every uncatalogued kind and the cell
  ;; relayed all of them. Nothing compared the two, so both suites stayed green.
  (doseq [k (concat uncatalogued ["move" "manipulate" "halt" "estop" "handback"])]
    (let [reasoner (try (ts/evaluate (ts/command k {:member-sig "sig:member"})
                                     (ts/grant {:force-auth-ref "forceauth:sbt/1"}))
                        :admitted
                        (catch #?(:clj Exception :cljs :default) _ :refused))
          cell     (try (sm/transition-relay-command
                         {"command_kind" k "member_sig" "sig:member"
                          "force_class" "soft-actuation" "force_auth_ref" "forceauth:sbt/1"})
                        :admitted
                        (catch #?(:clj Exception :cljs :default) _ :refused))]
      (is (= reasoner cell)
          (str "kind " (pr-str k) ": reasoner says " (name reasoner)
               " but the cell says " (name cell))))))

(deftest the-cell-refuses-an-uncatalogued-kind-with-the-catalogue-reason
  (doseq [k ["weaponize" "detonate" ""]]
    (let [e (try (sm/transition-relay-command
                  {"command_kind" k "member_sig" "sig:member"
                   "force_class" "soft-actuation" "force_auth_ref" "forceauth:sbt/1"})
                 nil
                 (catch #?(:clj clojure.lang.ExceptionInfo :cljs :default) e e))]
      (is (some? e) (str "the cell must refuse " (pr-str k)))
      (is (= :uncatalogued-command (:code (ex-data e))))
      (is (str/includes? (ex-message e) "not in the teleop catalogue")))))

(deftest the-cell-refuses-an-unauthorized-actuation
  ;; Entering the cell directly at relay used to skip force authorization entirely.
  (let [e (try (sm/transition-relay-command
                {"command_kind" "move" "member_sig" "sig:member"
                 "force_class" "soft-actuation" "force_auth_ref" ""})
               nil
               (catch #?(:clj clojure.lang.ExceptionInfo :cljs :default) e e))]
    (is (some? e) "relaying without force authorization must be refused")
    (is (= :not-force-authorized (:code (ex-data e))))))

(deftest the-cell-refuses-an-unrepresentable-force-class-at-relay
  (let [e (try (sm/transition-relay-command
                {"command_kind" "move" "member_sig" "sig:member"
                 "force_class" "weaponizable" "force_auth_ref" "forceauth:sbt/1"})
               nil
               (catch #?(:clj clojure.lang.ExceptionInfo :cljs :default) e e))]
    (is (some? e) "relaying under an unrepresentable force class must be refused")
    (is (= :unrepresentable-force-class (:code (ex-data e))))))
