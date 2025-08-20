# Δ135 v135.7-RKR — auto-repin + Rekor-seal: patch + sealed run (minimal console)
from pathlib import Path
from datetime import datetime, timezone
import json, os, subprocess, textwrap

ROOT = Path.cwd()
PROJ = ROOT / "truthlock"
SCRIPTS = PROJ / "scripts"
GUI = PROJ / "gui"
OUT = PROJ / "out"
SCHEMAS = PROJ / "schemas"
for d in (SCRIPTS, GUI, OUT, SCHEMAS): d.mkdir(parents=True, exist_ok=True)

# --- (1) Runner patch: auto-repin missing/invalid CIDs, write-back scroll, Rekor JSON proof ---
trigger = textwrap.dedent(r'''
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Δ135_TRIGGER — Initiate → Expand → Seal

- Scans truthlock/out/ΔLEDGER for sealed objects
- Validates ledger files (built-in + JSON Schema at truthlock/schemas/ledger.schema.json if jsonschema is installed)
- Guardrails for resolver: --max-bytes (env RESOLVER_MAX_BYTES), --allow (env RESOLVER_ALLOW or RESOLVER_ALLOW_GLOB),
  --deny (env RESOLVER_DENY or RESOLVER_DENY_GLOB)
- Auto-repin: missing or invalid CIDs get pinned (ipfs add -Q → fallback Pinata) and written back into the scroll JSON
- Emits ΔMESH_EVENT_135.json on --execute
- Optional: Pin Δ135 artifacts and Rekor-seal report
- Rekor: uploads report hash with --format json (if rekor-cli available), stores rekor_proof_<REPORT_SHA>.json
- Emits QR for best CID (report → trigger → any scanned)
"""
from __future__ import annotations
import argparse, hashlib, json, os, subprocess, sys, fnmatch, re
from datetime import datetime, timezone
from pathlib import Path
from typing import Any, Dict, List, Optional, Tuple

ROOT = Path.cwd()
OUTDIR = ROOT / "truthlock" / "out"
LEDGER_DIR = OUTDIR / "ΔLEDGER"
GLYPH_PATH = OUTDIR / "Δ135_GLYPH.json"
REPORT_PATH = OUTDIR / "Δ135_REPORT.json"
TRIGGER_PATH = OUTDIR / "Δ135_TRIGGER.json"
MESH_EVENT_PATH = OUTDIR / "ΔMESH_EVENT_135.json"
VALIDATION_PATH = OUTDIR / "ΔLEDGER_VALIDATION.json"
SCHEMA_PATH = ROOT / "truthlock" / "schemas" / "ledger.schema.json"

CID_PATTERN = re.compile(r'^(Qm[1-9A-HJ-NP-Za-km-z]{44,}|baf[1-9A-HJ-NP-Za-km-z]{20,})$')

def now_iso() -> str:
    return datetime.now(timezone.utc).replace(microsecond=0).isoformat()

def sha256_path(p: Path) -> str:
    h = hashlib.sha256()
    with p.open("rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()

def which(bin_name: str) -> Optional[str]:
    from shutil import which as _which
    return _which(bin_name)

def load_json(p: Path) -> Optional[Dict[str, Any]]:
    try:
        return json.loads(p.read_text(encoding="utf-8"))
    except Exception:
        return None

def write_json(path: Path, obj: Dict[str, Any]) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps(obj, ensure_ascii=False, indent=2), encoding="utf-8")

def find_ledger_objects() -> List[Path]:
    if not LEDGER_DIR.exists(): return []
    return sorted([p for p in LEDGER_DIR.glob("**/*.json") if p.is_file()])

# ---------- Guardrails ----------
def split_globs(s: str) -> List[str]:
    return [g.strip() for g in (s or "").split(",") if g.strip()]

def allowed_by_globs(rel_path: str, allow_globs: List[str], deny_globs: List[str]) -> Tuple[bool, str]:
    for g in deny_globs:
        if fnmatch.fnmatch(rel_path, g): return (False, f"denied by pattern: {g}")
    if allow_globs:
        for g in allow_globs:
            if fnmatch.fnmatch(rel_path, g): return (True, f"allowed by pattern: {g}")
        return (False, "no allowlist pattern matched")
    return (True, "no allowlist; allowed")

# ---------- Pin helpers ----------
def ipfs_add_cli(path: Path) -> Optional[str]:
    ipfs_bin = which("ipfs")
    if not ipfs_bin: return None
    try:
        return subprocess.check_output([ipfs_bin, "add", "-Q", str(path)], text=True).strip() or None
    except Exception:
        return None

def pinata_pin_json(obj: Dict[str, Any], name: str) -> Optional[str]:
    jwt = os.getenv("PINATA_JWT")
    if not jwt: return None
    token = jwt if jwt.startswith("Bearer ") else f"Bearer {jwt}"
    try:
        import urllib.request
        payload = {"pinataOptions": {"cidVersion": 1}, "pinataMetadata": {"name": name}, "pinataContent": obj}
        data = json.dumps(payload, ensure_ascii=False).encode("utf-8")
        req = urllib.request.Request("https://api.pinata.cloud/pinning/pinJSONToIPFS", data=data,
                                     headers={"Authorization": token, "Content-Type": "application/json"}, method="POST")
        with urllib.request.urlopen(req, timeout=30) as resp:
            info = json.loads(resp.read().decode("utf-8") or "{}")
            return info.get("IpfsHash") or info.get("ipfsHash")
    except Exception:
        return None

def maybe_pin_file_or_json(path: Path, obj: Optional[Dict[str, Any]], label: str) -> Tuple[str, str]:
    cid = None
    if path.exists():
        cid = ipfs_add_cli(path)
        if cid: return ("ipfs", cid)
    if obj is not None:
        cid = pinata_pin_json(obj, label)
        if cid: return ("pinata", cid)
    return ("pending", "")

# ---------- Rekor ----------
def rekor_upload_json(path: Path) -> Tuple[bool, Dict[str, Any]]:
    binp = which("rekor-cli")
    rep_sha = sha256_path(path)
    proof_path = OUTDIR / f"rekor_proof_{rep_sha}.json"
    if not binp:
        return (False, {"message": "rekor-cli not found", "proof_path": None})
    try:
        out = subprocess.check_output([binp, "upload", "--artifact", str(path), "--format", "json"],
                                      text=True, stderr=subprocess.STDOUT)
        try:
            data = json.loads(out)
        except Exception:
            data = {"raw": out}
        proof_path.write_text(json.dumps(data, ensure_ascii=False, indent=2), encoding="utf-8")
        info = {
            "ok": True,
            "uuid": data.get("UUID") or data.get("uuid"),
            "logIndex": data.get("LogIndex") or data.get("logIndex"),
            "proof_path": str(proof_path.relative_to(ROOT)),
            "raw": data
        }
        return (True, info)
    except subprocess.CalledProcessError as e:
        return (False, {"message": (e.output or "").strip(), "proof_path": None})
    except Exception as e:
        return (False, {"message": str(e), "proof_path": None})

# ---------- Validation ----------
def validate_builtin(obj: Dict[str, Any]) -> List[str]:
    errors: List[str] = []
    if not isinstance(obj, dict): return ["not a JSON object"]
    if not isinstance(obj.get("scroll_name"), str) or not obj.get("scroll_name"):
        errors.append("missing/invalid scroll_name")
    if "status" in obj and not isinstance(obj["status"], str):
        errors.append("status must be string if present")
    cid = obj.get("cid") or obj.get("ipfs_pin")
    if cid and not CID_PATTERN.match(str(cid)):
        errors.append("cid/ipfs_pin does not look like IPFS CID")
    return errors

def validate_with_schema(obj: Dict[str, Any]) -> List[str]:
    if not SCHEMA_PATH.exists(): return []
    try:
        import jsonschema
        schema = json.loads(SCHEMA_PATH.read_text(encoding="utf-8"))
        validator = getattr(jsonschema, "Draft202012Validator", jsonschema.Draft7Validator)(schema)
        return [f"{'/'.join([str(p) for p in e.path]) or '<root>'}: {e.message}" for e in validator.iter_errors(obj)]
    except Exception:
        return []

def write_validation_report(results: List[Dict[str, Any]]) -> Path:
    write_json(VALIDATION_PATH, {"timestamp": now_iso(), "results": results})
    return VALIDATION_PATH

# ---------- QR ----------
def emit_cid_qr(cid: Optional[str]) -> Dict[str, Optional[str]]:
    out = {"cid": cid, "png": None, "txt": None}
    if not cid: return out
    txt_path = OUTDIR / f"cid_{cid}.txt"
    txt_path.write_text(f"ipfs://{cid}\nhttps://ipfs.io/ipfs/{cid}\n", encoding="utf-8")
    out["txt"] = str(txt_path.relative_to(ROOT))
    try:
        import qrcode
        img = qrcode.make(f"ipfs://{cid}")
        png_path = OUTDIR / f"cid_{cid}.png"
        img.save(png_path)
        out["png"] = str(png_path.relative_to(ROOT))
    except Exception:
        pass
    return out

# ---------- Glyph ----------
def update_glyph(plan: Dict[str, Any], mode: str, pins: Dict[str, Dict[str, str]], extra: Dict[str, Any]) -> Dict[str, Any]:
    glyph = {
        "scroll_name": "Δ135_TRIGGER",
        "timestamp": now_iso(),
        "initiator": plan.get("initiator", "Matthew Dewayne Porter"),
        "meaning": "Initiate → Expand → Seal",
        "phases": plan.get("phases", ["ΔSCAN_LAUNCH","ΔMESH_BROADCAST_ENGINE","ΔSEAL_ALL"]),
        "summary": {
            "ledger_files": plan.get("summary", {}).get("ledger_files", 0),
            "unresolved_cids": plan.get("summary", {}).get("unresolved_cids", 0)
        },
        "inputs": plan.get("inputs", [])[:50],
        "last_run": {"mode": mode, **extra, "pins": pins}
    }
    write_json(GLYPH_PATH, glyph); return glyph

# ---------- Main ----------
def main(argv: Optional[List[str]] = None) -> int:
    ap = argparse.ArgumentParser(description="Δ135 auto-executing trigger")
    ap.add_argument("--dry-run", action="store_true")
    ap.add_argument("--execute", action="store_true")
    ap.add_argument("--resolve-missing", action="store_true")
    ap.add_argument("--pin", action="store_true")
    ap.add_argument("--rekor", action="store_true")
    ap.add_argument("--max-bytes", type=int, default=int(os.getenv("RESOLVER_MAX_BYTES", "10485760")))
    # env harmonization
    allow_env = os.getenv("RESOLVER_ALLOW", os.getenv("RESOLVER_ALLOW_GLOB", ""))
    deny_env  = os.getenv("RESOLVER_DENY",  os.getenv("RESOLVER_DENY_GLOB",  ""))
    ap.add_argument("--allow", action="append", default=[g for g in allow_env.split(",") if g.strip()])
    ap.add_argument("--deny",  action="append", default=[g for g in deny_env.split(",")  if g.strip()])
    args = ap.parse_args(argv)

    OUTDIR.mkdir(parents=True, exist_ok=True); LEDGER_DIR.mkdir(parents=True, exist_ok=True)

    # Scan ledger
    scanned: List[Dict[str, Any]] = []
    for p in find_ledger_objects():
        meta = {"path": str(p.relative_to(ROOT)), "size": p.stat().st_size, "mtime": int(p.stat().st_mtime)}
        j = load_json(p)
        if j:
            meta["scroll_name"] = j.get("scroll_name"); meta["status"] = j.get("status")
            meta["cid"] = j.get("cid") or j.get("ipfs_pin") or ""
        scanned.append(meta)

    # Validate
    validation_results: List[Dict[str, Any]] = []
    for item in scanned:
        j = load_json(ROOT / item["path"]) or {}
        errs = validate_with_schema(j) or validate_builtin(j)
        if errs: validation_results.append({"path": item["path"], "errors": errs})
    validation_report_path = write_validation_report(validation_results)

    # unresolved = missing OR invalid CID
    def is_invalid_or_missing(x): 
        c = x.get("cid", "")
        return (not c) or (not CID_PATTERN.match(str(c)))
    unresolved = [s for s in scanned if is_invalid_or_missing(s)]

    plan = {
        "scroll_name": "Δ135_TRIGGER", "timestamp": now_iso(),
        "initiator": os.getenv("GODKEY_IDENTITY", "Matthew Dewayne Porter"),
        "phases": ["ΔSCAN_LAUNCH", "ΔMESH_BROADCAST_ENGINE", "ΔSEAL_ALL"],
        "summary": {"ledger_files": len(scanned), "unresolved_cids": len(unresolved)},
        "inputs": scanned
    }
    write_json(TRIGGER_PATH, plan)

    if args.dry_run or (not args.execute):
        write_json(REPORT_PATH, {
            "timestamp": now_iso(), "mode": "plan",
            "plan_path": str(TRIGGER_PATH.relative_to(ROOT)),
            "plan_sha256": sha256_path(TRIGGER_PATH),
            "validation_report": str(validation_report_path.relative_to(ROOT)),
            "result": {"message": "Δ135 planning only (no actions executed)"}
        })
        update_glyph(plan, mode="plan", pins={}, extra={
            "report_path": str(REPORT_PATH.relative_to(ROOT)),
            "report_sha256": sha256_path(REPORT_PATH),
            "mesh_event_path": None,
            "qr": {"cid": None}
        })
        print(f"[Δ135] Planned. Ledger files={len(scanned)} unresolved_cids={len(unresolved)}")
        return 0

    # Resolve (auto-repin) with guardrails; write-back scroll JSON on success
    cid_resolution: List[Dict[str, Any]] = []
    if args.resolve_missing and unresolved:
        allow_globs = [g for sub in (args.allow or []) for g in (split_globs(sub) or [""]) if g]
        deny_globs  = [g for sub in (args.deny  or []) for g in (split_globs(sub) or [""]) if g]
        for item in list(unresolved):
            rel = item["path"]; ledger_path = ROOT / rel
            # guardrails
            ok, reason = allowed_by_globs(rel, allow_globs, deny_globs)
            if not ok:
                cid_resolution.append({"path": rel, "action": "skip", "reason": reason}); continue
            if (not ledger_path.exists()) or (ledger_path.stat().st_size > args.max_bytes):
                cid_resolution.append({"path": rel, "action": "skip", "reason": f"exceeds max-bytes ({args.max_bytes}) or missing"}); continue
            # pin flow
            j = load_json(ledger_path) or {}
            prev = j.get("cid")
            mode, cid = maybe_pin_file_or_json(ledger_path, j, f"ΔLEDGER::{ledger_path.name}")
            if cid:
                j["cid"] = cid  # write back
                try: ledger_path.write_text(json.dumps(j, ensure_ascii=False, indent=2), encoding="utf-8")
                except Exception: pass
                item["cid"] = cid
                cid_resolution.append({"path": rel, "action": "repinned", "mode": mode, "prev": prev, "cid": cid})
        # recompute unresolved
        unresolved = [s for s in scanned if (not s.get("cid")) or (not CID_PATTERN.match(str(s.get("cid",""))))]
        plan["summary"]["unresolved_cids"] = len(unresolved)
        write_json(TRIGGER_PATH, plan)

    # Mesh event
    affected = [{"path": i["path"], "cid": i.get("cid", ""), "scroll_name": i.get("scroll_name")} for i in scanned]
    event = {"event_name": "ΔMESH_EVENT_135", "timestamp": now_iso(), "trigger": "Δ135",
             "affected": affected, "actions": ["ΔSCAN_LAUNCH","ΔMESH_BROADCAST_ENGINE","ΔSEAL_ALL"]}
    write_json(MESH_EVENT_PATH, event)

    pins: Dict[str, Dict[str, str]] = {}
    if args.pin:
        mode, ident = maybe_pin_file_or_json(TRIGGER_PATH, plan, "Δ135_TRIGGER")
        pins["Δ135_TRIGGER"] = {"mode": mode, "id": ident}

    # Best CID + QR
    best_cid = pins.get("Δ135_REPORT", {}).get("id") if pins else None
    if not best_cid: best_cid = pins.get("Δ135_TRIGGER", {}).get("id") if pins else None
    if not best_cid:
        for s in scanned:
            if s.get("cid"): best_cid = s["cid"]; break
    qr = emit_cid_qr(best_cid)

    # Report
    result = {"timestamp": now_iso(), "mode": "execute",
              "mesh_event_path": str(MESH_EVENT_PATH.relative_to(ROOT)),
              "mesh_event_hash": sha256_path(MESH_EVENT_PATH)}
    report = {"timestamp": now_iso(), "plan": plan, "event": event, "result": result,
              "pins": pins, "cid_resolution": cid_resolution,
              "validation_report": str(validation_report_path.relative_to(ROOT)), "qr": qr}
    write_json(REPORT_PATH, report)

    # Rekor sealing (optional)
    if args.rekor:
        ok, info = rekor_upload_json(REPORT_PATH)
        report["rekor"] = {"ok": ok, **info}
        write_json(REPORT_PATH, report)

    # Pin the report (optional, after Rekor for stable hash capture)
    if args.pin:
        rep_obj = load_json(REPORT_PATH)
        mode, ident = maybe_pin_file_or_json(REPORT_PATH, rep_obj, "Δ135_REPORT")
        pins["Δ135_REPORT"] = {"mode": mode, "id": ident}
        report["pins"] = pins; write_json(REPORT_PATH, report)

    # Glyph
    extra = {"report_path": str(REPORT_PATH.relative_to(ROOT)),
             "report_sha256": sha256_path(REPORT_PATH),
             "mesh_event_path": str(MESH_EVENT_PATH.relative_to(ROOT)),
             "qr": qr}
    if report.get("rekor", {}).get("proof_path"):
        extra["rekor_proof"] = report["rekor"]["proof_path"]
        extra["rekor_uuid"] = report["rekor"].get("uuid")
        extra["rekor_logIndex"] = report["rekor"].get("logIndex")
    update_glyph(plan, mode="execute", pins=pins, extra=extra)

    print(f"[Δ135] Executed. Mesh event → {MESH_EVENT_PATH.name}")
    return 0

if __name__ == "__main__":
    sys.exit(main())
''').strip("\n")

(SCRIPTS / "Δ135_TRIGGER.py").write_text(trigger, encoding="utf-8")

# --- (2) Dashboard patch: Rekor panel + pinning matrix ---
tile = textwrap.dedent(r'''
import json, os, subprocess
from pathlib import Path
import streamlit as st

ROOT = Path.cwd()
OUTDIR = ROOT / "truthlock" / "out"
GLYPH = OUTDIR / "Δ135_GLYPH.json"
REPORT = OUTDIR / "Δ135_REPORT.json"
TRIGGER = OUTDIR / "Δ135_TRIGGER.json"
EVENT = OUTDIR / "ΔMESH_EVENT_135.json"
VALID = OUTDIR / "ΔLEDGER_VALIDATION.json"

def load_json(p: Path):
    try: return json.loads(p.read_text(encoding="utf-8"))
    except Exception: return {}

st.title("Δ135 — Auto-Repin + Rekor")
st.caption("Initiate → Expand → Seal  •  ΔSCAN_LAUNCH → ΔMESH_BROADCAST_ENGINE → ΔSEAL_ALL")

glyph = load_json(GLYPH)
report = load_json(REPORT)
plan = load_json(TRIGGER)
validation = load_json(VALID)

c1, c2, c3, c4 = st.columns(4)
c1.metric("Ledger files", plan.get("summary", {}).get("ledger_files", 0))
c2.metric("Unresolved CIDs", plan.get("summary", {}).get("unresolved_cids", 0))
c3.metric("Last run", (glyph.get("last_run", {}) or {}).get("mode", (report or {}).get("mode", "—")))
c4.metric("Timestamp", glyph.get("timestamp", "—"))

issues = validation.get("results", [])
if isinstance(issues, list) and len(issues) == 0:
    st.success("Ledger validation: clean ✅")
else:
    st.error(f"Ledger validation: {len(issues)} issue(s) ❗")
    with st.expander("Validation details"): st.json(issues)

with st.expander("Guardrails (env)"):
    st.write("**Max bytes:**", os.getenv("RESOLVER_MAX_BYTES", "10485760"))
    st.write("**Allow globs:**", os.getenv("RESOLVER_ALLOW", os.getenv("RESOLVER_ALLOW_GLOB", "")) or "—")
    st.write("**Deny globs:**",  os.getenv("RESOLVER_DENY",  os.getenv("RESOLVER_DENY_GLOB",  "")) or "—")

st.write("---")
st.subheader("Rekor Transparency")
rk = (report or {}).get("rekor", {})
if rk.get("ok"):
    st.success("Rekor sealed ✅")
    st.write("UUID:", rk.get("uuid") or "—")
    st.write("Log index:", rk.get("logIndex") or "—")
    if rk.get("proof_path"):
        proof = ROOT / rk["proof_path"]
        if proof.exists():
            st.download_button("Download Rekor proof", proof.read_bytes(), file_name=proof.name)
else:
    st.info(rk.get("message") or "Not sealed (run with --rekor)")

st.write("---")
st.subheader("Pinning Matrix")
rows = []
for r in (report.get("cid_resolution") or []):
    rows.append({"path": r.get("path"), "action": r.get("action"), "mode": r.get("mode"),
                 "cid": r.get("cid"), "reason": r.get("reason")})
if rows:
    st.dataframe(rows, hide_index=True)
else:
    st.caption("No CID resolution activity in last run.")

st.write("---")
st.subheader("Run Controls")
with st.form("run135"):
    a,b,c,d = st.columns(4)
    execute = a.checkbox("Execute", True)
    resolve = b.checkbox("Resolve missing", True)
    pin     = c.checkbox("Pin artifacts", True)
    rekor   = d.checkbox("Rekor upload", True)
    max_bytes = st.number_input("Max bytes", value=int(os.getenv("RESOLVER_MAX_BYTES","10485760")), min_value=0, step=1_048_576)
    allow = st.text_input("Allow globs (comma-separated)", value=os.getenv("RESOLVER_ALLOW", os.getenv("RESOLVER_ALLOW_GLOB","")))
    deny  = st.text_input("Deny globs (comma-separated)",  value=os.getenv("RESOLVER_DENY",  os.getenv("RESOLVER_DENY_GLOB","")))
    go = st.form_submit_button("Run Δ135")
    if go:
        args = []
        if execute: args += ["--execute"]
        else: args += ["--dry-run"]
        if resolve: args += ["--resolve-missing"]
        if pin: args += ["--pin"]
        if rekor: args += ["--rekor"]
        args += ["--max-bytes", str(int(max_bytes))]
        if allow.strip():
            for a1 in allow.split(","):
                a1=a1.strip()
                if a1: args += ["--allow", a1]
        if deny.strip():
            for d1 in deny.split(","):
                d1=d1.strip()
                if d1: args += ["--deny", d1]
        subprocess.call(["python", "truthlock/scripts/Δ135_TRIGGER.py", *args])
        st.experimental_rerun()

st.write("---")
st.subheader("Latest CID & QR")
qr = (glyph.get("last_run", {}) or {}).get("qr") or (report or {}).get("qr") or {}
if qr.get("cid"):
    st.write(f"CID: `{qr['cid']}`")
    png = OUTDIR / f"cid_{qr['cid']}.png"
    txt = OUTDIR / f"cid_{qr['cid']}.txt"
    if png.exists():
        st.image(str(png), caption=f"QR for ipfs://{qr['cid']}")
        st.download_button("Download QR PNG", png.read_bytes(), file_name=png.name)
    if txt.exists():
        st.download_button("Download QR TXT", txt.read_bytes(), file_name=txt.name)
else:
    st.caption("No CID yet.")

st.write("---")
st.subheader("Artifacts")
cols = st.columns(4)
if TRIGGER.exists(): cols[0].download_button("Δ135_TRIGGER.json", TRIGGER.read_bytes(), file_name="Δ135_TRIGGER.json")
if REPORT.exists():  cols[1].download_button("Δ135_REPORT.json",  REPORT.read_bytes(),  file_name="Δ135_REPORT.json")
if EVENT.exists():   cols[2].download_button("ΔMESH_EVENT_135.json", EVENT.read_bytes(), file_name="ΔMESH_EVENT_135.json")
if VALID.exists():   cols[3].download_button("ΔLEDGER_VALIDATION.json", VALID.read_bytes(), file_name="ΔLEDGER_VALIDATION.json")
''').strip("\n")

(GUI / "Δ135_tile.py").write_text(tile, encoding="utf-8")

# --- (3) Execute sealed run (uses env if present) ---
def run(cmd): 
    p = subprocess.run(cmd, cwd=str(ROOT), capture_output=True, text=True)
    return p.returncode, p.stdout.strip(), p.stderr.strip()

rc, out, err = run([
    "python", str(SCRIPTS / "Δ135_TRIGGER.py"),
    "--execute", "--resolve-missing", "--pin", "--rekor",
    "--max-bytes", "10485760", "--allow", "truthlock/out/ΔLEDGER/*.json"
])

# Write a tiny summary for quick inspection
summary = {
    "ts": datetime.now(timezone.utc).replace(microsecond=0).isoformat(),
    "rc": rc, "stdout": out, "stderr": err,
    "artifacts": sorted(p.name for p in OUT.iterdir())
}
(OUT / "Δ135_RKR_SUMMARY.json").write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding="utf-8")
print(json.dumps(summary, ensure_ascii=False))packages/starlight/README.md