(ns tazuna.operation
  "tazuna 手綱 — the teleoperation command catalogue (the actor's `:operation` part).

  ## Why this namespace exists

  Before it, the admitted teleop vocabulary was written down FOUR times and the
  four copies disagreed:

    1. `lex/teleopCommand.edn`      — `kind` enum, the on-wire authority
    2. `methods/teleop_safety.kotoba` — the canonical safety kernel, command-kind 0..4
    3. `methods/teleop_safety.cljc` — `ACTUATION-KINDS`, refuses an unknown kind
    4. `cells/teleop_session/state_machine.cljc` — **no allowlist at all**

  Measured 2026-09-06 against f180a77: (4) relayed `\"weaponize\"`, `\"detonate\"`
  and `\"\"` as member-signed actuation with `onChainAnchored true`, while (2) and
  (3) refuse every one of them and (1) cannot even represent them. The gate was
  real in three places and absent in the fourth — and the fourth is the one the
  cell actually relays through.

  So the catalogue is stated ONCE here, and `tazuna.governor` adjudicates against
  it. This is a projection of the lexicon, not a fifth opinion: the parity is
  pinned by `tazuna.governor-test/the-catalogue-is-the-lexicon-vocabulary`, which
  reads `lex/teleopCommand.edn` and fails if the two ever drift.

  ## Deny-by-default

  A kind that is not catalogued is refused. That is the whole point: N1 says a
  weaponizable / force-as-harm command is *structurally unrepresentable*, and a
  vocabulary that admits whatever it is handed cannot make that claim.

  Pure + portable (`.cljc`): no I/O, no host types, so it runs wherever the
  reasoner and the state machine run."
  (:require [kotoba.lang.text :as str]))

;; ── the catalogue ────────────────────────────────────────────────────────────
;; :kernel-code — the integer this kind carries across the `methods/
;;   teleop_safety.kotoba` boundary. Kept here so the stable protocol has exactly
;;   one definition on the Clojure side too.
;; :actuates? — does honouring this command move a physical machine? Only an
;;   actuating command needs a member signature (G4) and a force authorization
;;   (G3); a safety command is the thing you reach for when those have gone wrong.
;; :always-permitted? — halt / estop / handback are never gated and never signed.
;;   Stated by the README, the state machine, and the lexicon description alike.
(def catalogue
  {"halt"       {:kernel-code 0 :actuates? false :always-permitted? true
                 :description "stop moving; hold position"}
   "estop"      {:kernel-code 1 :actuates? false :always-permitted? true
                 :description "emergency stop; always honoured"}
   "handback"   {:kernel-code 2 :actuates? false :always-permitted? true
                 :description "return control to the autonomy stack"}
   "move"       {:kernel-code 3 :actuates? true  :always-permitted? false
                 :description "locomote the robot base"}
   "manipulate" {:kernel-code 4 :actuates? true  :always-permitted? false
                 :description "articulate an end effector"}})

;; N1 — the benign force classes. The absence of a kinetic/weapon class is the
;; invariant; `weaponizable` is refused because it is not here, not because it
;; is named anywhere as forbidden.
(def force-classes #{"observational" "soft-actuation" "powered-actuation"})

(defn command-kinds
  "Every catalogued command kind. Pinned against the lexicon enum by the tests."
  []
  (set (keys catalogue)))

(defn permitted?
  "Deny-by-default: true only for a kind this catalogue actually names."
  [kind]
  (contains? catalogue kind))

(defn actuates?
  "Does this kind move a physical machine? False for anything uncatalogued —
  an unknown kind is refused before this question is ever interesting."
  [kind]
  (boolean (get-in catalogue [kind :actuates?])))

(defn always-permitted?
  "Safety commands (halt / estop / handback) are never gated and never signed."
  [kind]
  (boolean (get-in catalogue [kind :always-permitted?])))

(defn representable-force-class?
  "N1: only the three benign classes are representable."
  [force-class]
  (contains? force-classes force-class))

(defn kernel-code
  "The integer this kind carries across the `teleop_safety.kotoba` boundary."
  [kind]
  (get-in catalogue [kind :kernel-code]))

(defn refusal-detail
  "Why an uncatalogued kind is refused, naming the vocabulary it is missing from.

  The wording is specific on purpose: a refusal that says only \"refused\" cannot
  be told apart from a refusal for some other reason, so a test asserting
  non-emptiness proves nothing (the failure mode this loop hit on 2026-09-05)."
  [kind]
  (str "N1: command kind " (pr-str kind) " is not in the teleop catalogue "
       "{" (str/join ", " (sort (command-kinds))) "}; an uncatalogued command is "
       "structurally unrepresentable and is never relayed (Mission Charter §1.12, "
       "lex/teleopCommand.edn)"))
