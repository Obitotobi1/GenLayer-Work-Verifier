# Work Verifier — GenLayer Intelligent Contract

Work Verifier evaluates one worker's written submission against a checklist fixed when the contract is deployed. Each requirement receives `PASS`, `FAIL`, or `UNKNOWN`. Validators independently re-evaluate the same input and must agree on every checklist outcome before the contract stores an attempt.

The final status is derived from the accepted outcomes:

- `APPROVED`: every requirement is `PASS`; the task is then closed.
- `REJECTED`: at least one requirement is `FAIL`; the worker may revise and resubmit.
- `NEEDS_REVIEW`: no requirement is `FAIL`, but at least one is `UNKNOWN`; the worker may revise and resubmit. This status does not create a human-review process.

The contract does not transfer funds or verify external actions.

## Contract behavior

- The deployer becomes the owner; the constructor assigns one worker.
- The title and 1–8 requirements are immutable after deployment.
- Only the assigned worker can submit text, with a maximum of 16,000 characters.
- A contract stores at most five consensus-approved attempts. Failed transactions and validator disagreement do not consume an attempt.
- Each attempt stores the submission, derived status, per-requirement outcomes, and reasons.
- `get_task()` returns task metadata and current status; `get_attempt(index)` returns one stored record using zero-based indexes.
- Model output is size-limited and validated for exact JSON shape, valid outcomes, and bounded reasons before storage.
- The task title, checklist, and submission are placed in the prompt as JSON data and explicitly treated as untrusted. This reduces prompt-injection risk but cannot guarantee model resistance.

## Run local tests

From this directory:

```sh
python -m unittest test_contract -v
```

These tests use a small mock of the GenLayer SDK. They exercise contract logic such as access control, status derivation, retry limits, malformed model responses, and validator disagreement. They do **not** run GenVM, real validators, live AI, storage serialization, or a deployed network.

For GenLayer's SDK-based validation and ABI/schema extraction, install the official linter and run:

```sh
pip install genvm-linter
genvm-lint check work_verifier.py
genvm-lint schema work_verifier.py
```

## Try it in GenLayer Studio

1. Open GenLayer Studio and create or load `work_verifier.py`.
2. Deploy with constructor values like these, using the wallet address that will submit the work:

```json
{
  "title": "A short planting guide",
  "requirements": [
    "Explain that seeds should be planted in soil.",
    "Explain that seedlings need sunlight and regular watering."
  ],
  "worker": "REPLACE_WITH_WORKER_WALLET_ADDRESS"
}
```

3. From the assigned worker wallet, call `submit_work` with a written submission.
4. Wait for the transaction to reach finality, then inspect `get_attempt(0)` and `get_task()`.

This file uses the pinned runtime identifier in the current official GenLayer contract documentation. Confirm that the target network supports it before deployment.

Live GenLayer Studio tests have confirmed all three outcomes:

- `APPROVED`: complete submission, both checklist outcomes `PASS`.
- `REJECTED`: one requirement `PASS`, one `FAIL` (submission omitted sunlight and watering). Contract: `0xd26Db41C34987C8Af4A5953fD3d3F0b8DD4C1dC1`.
- `NEEDS_REVIEW`: requirement "Publish this on my website." came back `UNKNOWN` because the external action cannot be verified from text alone. Contract: `0xa04658A716542B422d28A25482C71B009DB14F8C`.

A live contract result does not guarantee that a contribution will be accepted or receive points.

## Design limits

- The contract evaluates text only. It does not fetch URLs or inspect files, images, publication, identity, payment, authorship, or real-world completion.
- Checklist reasons are included from the accepted leader result; validators agree on the outcome for each item, not identical wording of explanations.
- `NEEDS_REVIEW` means the text did not establish a verdict. It does not assign a reviewer or resolve a dispute.
- Consensus, provider, or network failures can prevent a transaction from completing.
- All submitted text and stored assessments are public contract data. Do not submit private information.
