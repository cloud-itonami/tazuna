(ns tazuna.governor
  "tazuna 手綱 — the independent teleop Governor (the actor's `:governor` part).

  ## What it is for

  The build-actor pattern puts the advisor behind a Governor that can refuse it.
  tazuna had the refusals but not the Governor: each transition in
  `cells/teleop_session/state_machine.cljc` carried its own inline checks, so a
  transition was only as safe as the checks it happened to contain, and
  `transition-relay-command` — the one that actually relays actuation — trusted
  that some earlier transition had run.

  Measured 2026-09-06 against f180a77, entering directly at relay:

    kind \"weaponize\" / \"detonate\" / \"\"  → RELAYED, onChainAnchored true
    force_auth_ref \"\" (never authorized)  → RELAYED
    force_class \"weaponizable\"            → RELAYED

  All three are refused by `methods/teleop_safety.kotoba`, the repo's own
  canonical safety kernel, which returns 14 / 11 / 10 for exactly these. The
  Governor makes the cell agree with the kernel it already ships.

  ## Deny-by-default

  `adjudicate` returns a decision for EVERY request. There is no fall-through
  admit: a request is admitted only by reaching the final clause with every gate
  satisfied, and each refusal names the gate that produced it (`:gate`), so a
  caller — and a test — can tell one refusal from another.

  Order mirrors the kernel's `authorization-code`, because a request that is
  inadmissible for two reasons should report the same one the kernel would:

    G4  server signature present        → :server-signature-refused   (kernel 12)
    N1  kind not in the catalogue       → :uncatalogued-command       (kernel 14)
    N1  force class unrepresentable     → :unrepresentable-force-class(kernel 10)
    G3  no force-authorization ref      → :not-force-authorized       (kernel 11)
    G4  actuation without member sig    → :member-signature-required  (kernel 13)
    —   admitted

  Safety commands (halt / estop / handback) are exempt from the N1 force-class,
  G3 and G4 signature gates by design: they are what you reach for when those
  gates have already failed, and the README, the state machine and the lexicon
  all say they are always honoured. They are NOT exempt from the catalogue — a
  kind nobody catalogued is not a safety command just because it was asserted to
  be one. That is the one gate that binds every request.

  Pure + portable (`.cljc`): a decision map in, a decision map out, no I/O."
  (:require [tazuna.operation :as op]))

(defn- refuse [gate code detail]
  {:decision :refuse :gate gate :code code :kernel-code (get {:server-signature-refused    12
                                                              :uncatalogued-command        14
                                                              :unrepresentable-force-class 10
                                                              :not-force-authorized        11
                                                              :member-signature-required   13}
                                                             code)
   :detail detail})

(defn adjudicate
  "Adjudicate one relay request. Returns

    {:decision :admit  :kind k :actuates? bool :safety? bool}
    {:decision :refuse :gate \"G3\" :code :not-force-authorized :kernel-code 11 :detail \"…\"}

  `request` keys (all optional; absent is treated as absent, never as satisfied):

    :kind :force-class :force-auth-ref :member-sig :server-sig"
  [{:keys [kind force-class force-auth-ref member-sig server-sig]}]
  (let [present? (fn [s] (boolean (seq (str s))))]
    (cond
      ;; G4 — the platform never signs a physical-robot command, for any kind.
      (present? server-sig)
      (refuse "G4" :server-signature-refused
              (str "G4: server signature refused — the platform never signs a "
                   "physical-robot command (no-server-key, ADR-2605231525)"))

      ;; N1 — deny-by-default on the vocabulary. Binds safety commands too.
      (not (op/permitted? kind))
      (refuse "N1" :uncatalogued-command (op/refusal-detail kind))

      ;; Safety commands are admitted here: never gated, never signed.
      (op/always-permitted? kind)
      {:decision :admit :kind kind :actuates? false :safety? true}

      ;; N1 — a session whose force class is unrepresentable can never actuate.
      (not (op/representable-force-class? force-class))
      (refuse "N1" :unrepresentable-force-class
              (str "N1: force class " (pr-str force-class) " is unrepresentable; "
                   "weaponizable / force-as-harm can never be admitted "
                   "(Mission Charter §1.12)"))

      ;; G3 — Transparent Force: no actuation without an authorizing reference.
      (not (present? force-auth-ref))
      (refuse "G3" :not-force-authorized
              (str "G3: Transparent Force requires a force-authorization reference "
                   "(1 SBT=1 vote admission) before an actuation command is relayed"))

      ;; G4 — the member signs every actuation command.
      (not (present? member-sig))
      (refuse "G4" :member-signature-required
              "G4: a member signature is required to relay an actuation command")

      :else
      {:decision :admit :kind kind :actuates? true :safety? false})))

(defn admitted?
  "True only for an explicit :admit. Anything else — including a malformed
  decision — is not an admission."
  [decision]
  (= :admit (:decision decision)))
