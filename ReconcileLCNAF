#!/usr/bin/env python3

import argparse
import csv
import json
import random
import re
import time
from copy import deepcopy
from urllib.parse import quote, urlparse

import requests
from asnake.client import ASnakeClient
from requests.exceptions import ConnectionError, HTTPError, Timeout


ID_LOC_NAME_RE = re.compile(
    r"^https?://id\.loc\.gov/authorities/names/[a-z0-9]+/?$",
    re.I,
)

OCCUPATIONAL_TITLES_RE = (
    r"Senator|Representative|Governor|President|Judge|Justice|"
    r"Dr\.?|Rev\.?|Professor"
)

HEADERS = {
    "Accept": "application/json",
    "User-Agent": "ArchivesSpace LCNAF agent reconciliation script",
}

LC_TIMEOUT = 60
LC_RETRIES = 3
LC_BACKOFF_BASE = 1.5

MADS = "http://www.loc.gov/mads/rdf/v1#"
SKOS = "http://www.w3.org/2004/02/skos/core#"

AGENT_ENDPOINTS = {
    "people": {
        "path": "agents/people",
        "name_jsonmodel_type": "name_person",
        "lc_types": {"PersonalName"},
    },
    "corporate_entities": {
        "path": "agents/corporate_entities",
        "name_jsonmodel_type": "name_corporate_entity",
        "lc_types": {"CorporateName", "ConferenceName"},
    },
    "families": {
        "path": "agents/families",
        "name_jsonmodel_type": "name_family",
        "lc_types": {"FamilyName"},
    },
}


def lc_get(url, *, params=None, allow_redirects=True, context="LC request"):
    last_exc = None

    for attempt in range(1, LC_RETRIES + 1):
        try:
            response = requests.get(
                url,
                params=params,
                headers=HEADERS,
                timeout=LC_TIMEOUT,
                allow_redirects=allow_redirects,
            )

            if response.status_code in {429, 500, 502, 503, 504}:
                raise HTTPError(
                    f"{context}: retryable status {response.status_code} for {response.url}",
                    response=response,
                )

            return response

        except (Timeout, ConnectionError, HTTPError) as exc:
            last_exc = exc

            if attempt == LC_RETRIES:
                break

            sleep_for = (LC_BACKOFF_BASE ** attempt) + random.uniform(0, 0.75)
            time.sleep(sleep_for)

    raise Timeout(f"{context} failed after {LC_RETRIES} attempts: {last_exc}")


def load_secrets(path="secrets.json"):
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)


def make_client(secrets):
    client = ASnakeClient(
        baseurl=secrets["baseurl"],
        username=secrets["username"],
        password=secrets["password"],
    )
    client.authorize()
    return client


def safe_json_response(response, context):
    text = response.text.strip()

    if not text:
        raise ValueError(f"Empty response from {context}: {response.url}")

    try:
        return response.json()
    except ValueError:
        raise ValueError(
            f"Non-JSON response from {context}: {response.url} "
            f"status={response.status_code} "
            f"content_type={response.headers.get('Content-Type')}"
        )


def clean_spaces(value):
    return re.sub(r"\s+", " ", str(value or "")).strip()


def normalize_lc_uri(uri):
    uri = str(uri).strip()
    if uri.startswith("/"):
        uri = f"https://id.loc.gov{uri}"
    uri = uri.replace("https://", "http://").rstrip("/")
    uri = re.sub(r"[?#].*$", "", uri)
    uri = re.sub(r"\.(json|rdf|madsxml|marcxml|nt|xml)$", "", uri)
    return uri


def https_lc_uri(uri):
    return normalize_lc_uri(uri).replace("http://", "https://")


def is_lcnaf_uri(uri):
    return bool(ID_LOC_NAME_RE.match(str(uri or "").strip()))


def source_for_lc_uri(uri):
    path = urlparse(str(uri)).path
    if path.startswith("/authorities/names/"):
        return "lcnaf"
    return None


def as_list(value):
    if value is None:
        return []
    return value if isinstance(value, list) else [value]


def value_text(value):
    if isinstance(value, str):
        return value
    if isinstance(value, dict):
        return value.get("@value") or value.get("value") or ""
    return ""


def first_text(node, keys):
    for key in keys:
        for value in as_list(node.get(key)):
            text = value_text(value)
            if text:
                return text
    return ""


def extract_lcnaf_uris(value):
    found = []

    def scan(v):
        if isinstance(v, str):
            if "/authorities/names/" in v:
                found.append(v)
        elif isinstance(v, dict):
            for vv in v.values():
                scan(vv)
        elif isinstance(v, list):
            for vv in v:
                scan(vv)

    scan(value)

    clean = []
    seen = set()

    for uri in found:
        uri = normalize_lc_uri(uri)

        if source_for_lc_uri(uri) != "lcnaf":
            continue

        if uri not in seen:
            seen.add(uri)
            clean.append(uri)

    return clean


def has_id_loc_authority(agent):
    for name in agent.get("names", []):
        if is_lcnaf_uri(name.get("authority_id")):
            return True
    return False


def best_existing_name(agent):
    names = agent.get("names", [])
    display = next((n for n in names if n.get("is_display_name")), None)
    return display or names[0] if names else None


def get_lc_graph(uri):
    uri = normalize_lc_uri(uri)
    r = lc_get(f"{uri}.json", context="LCNAF authority detail")
    r.raise_for_status()
    data = safe_json_response(r, "LCNAF authority detail")

    if isinstance(data, dict):
        return data.get("@graph", [])
    if isinstance(data, list):
        return data

    raise ValueError(f"Unexpected JSON structure for {uri}")


def graph_node_by_id(graph, node_id):
    target = str(node_id or "")

    for node in graph:
        if not isinstance(node, dict):
            continue

        if str(node.get("@id", "")) == target:
            return node

    normalized_target = normalize_lc_uri(target)

    for node in graph:
        if not isinstance(node, dict):
            continue

        if normalize_lc_uri(node.get("@id", "")) == normalized_target:
            return node

    return {}


def get_target_node(graph, uri):
    return graph_node_by_id(graph, uri)


def get_authoritative_label(node):
    label = first_text(
        node,
        [
            f"{MADS}authoritativeLabel",
            "authoritativeLabel",
            f"{SKOS}prefLabel",
            "prefLabel",
        ],
    )

    if not label:
        raise ValueError("No authoritative LCNAF label found")

    return label


def get_mads_types(node):
    types = set()

    for value in as_list(node.get("@type")):
        if not isinstance(value, str):
            continue

        short = value.rsplit("#", 1)[-1].rsplit("/", 1)[-1]
        short = short.replace("madsrdf:", "")
        types.add(short)

    return types


def lc_type_matches_agent_type(types, agent_type):
    wanted = AGENT_ENDPOINTS[agent_type]["lc_types"]

    if not types:
        return True

    return bool(types & wanted)


def normalize_match_label(label):
    label = clean_spaces(label).casefold()
    label = re.sub(r"[.,;:]+", "", label)
    label = re.sub(r"\s+", " ", label)
    return label.strip()


def normalized_name_label(label):
    label = clean_spaces(label).casefold()
    label = label.replace("&", "and")
    label = re.sub(r"[.,;:]+", "", label)
    label = re.sub(r"\s+", " ", label)
    return label.strip()


def get_known_lcnaf_uri_by_label(label):
    if not label:
        return None

    encoded_label = quote(label.strip(), safe="")
    url = f"https://id.loc.gov/authorities/names/label/{encoded_label}"

    r = lc_get(
        url,
        allow_redirects=False,
        context="LCNAF known-label lookup",
    )

    if r.status_code in (301, 302, 303, 307, 308):
        location = r.headers.get("Location")
        if location:
            uris = extract_lcnaf_uris(location)
            if uris:
                return uris[0]

    r2 = lc_get(
        url,
        allow_redirects=True,
        context="LCNAF known-label redirect lookup",
    )

    if "/authorities/names/" in r2.url:
        uris = extract_lcnaf_uris(r2.url)
        if uris:
            return uris[0]

    if r.status_code == 200:
        try:
            data = safe_json_response(r, "LCNAF known-label lookup")
        except Exception:
            return None

        uris = extract_lcnaf_uris(data)
        if uris:
            return uris[0]

    return None


def get_lc_variant_labels(graph, authority_node):
    labels = []

    variant_keys = [
        f"{MADS}hasVariant",
        "hasVariant",
        f"{MADS}variantLabel",
        "variantLabel",
        f"{SKOS}altLabel",
        "altLabel",
    ]

    label_keys = [
        f"{MADS}variantLabel",
        "variantLabel",
        f"{SKOS}altLabel",
        "altLabel",
    ]

    def add_label(label):
        label = clean_spaces(label)
        if label:
            labels.append(label)

    for key in variant_keys:
        for value in as_list(authority_node.get(key)):
            if isinstance(value, dict):
                inline_label = first_text(value, label_keys)
                if inline_label:
                    add_label(inline_label)

                ref_id = value.get("@id")
                if ref_id:
                    ref_node = graph_node_by_id(graph, ref_id)
                    if ref_node:
                        ref_label = first_text(ref_node, label_keys)
                        if ref_label:
                            add_label(ref_label)
            else:
                text = value_text(value)
                if text:
                    add_label(text)

    unique = []
    seen = set()

    for label in labels:
        normalized = normalized_name_label(label)
        if normalized and normalized not in seen:
            seen.add(normalized)
            unique.append(label)

    return unique


def get_lcnaf_match_from_uri(uri, agent_type, method):
    graph = get_lc_graph(uri)
    node = get_target_node(graph, uri)

    if not node:
        return None

    types = get_mads_types(node)

    if not lc_type_matches_agent_type(types, agent_type):
        return None

    auth_label = get_authoritative_label(node)

    return {
        "uri": https_lc_uri(uri),
        "label": auth_label,
        "types": sorted(types),
        "method": method,
        "variant_labels": get_lc_variant_labels(graph, node),
    }


def get_lcnaf_search_matches(label, agent_type, count=10):
    url = "https://id.loc.gov/search/"
    params = {
        "q": label,
        "qf": "cs:http://id.loc.gov/authorities/names",
        "format": "json",
        "count": count,
    }

    r = lc_get(
        url,
        params=params,
        context="LCNAF search",
    )

    if r.status_code == 404:
        return None

    r.raise_for_status()
    data = safe_json_response(r, "LCNAF search")

    matches = []

    for uri in extract_lcnaf_uris(data):
        try:
            match = get_lcnaf_match_from_uri(uri, agent_type, "idloc_search")
        except Timeout:
            raise
        except Exception:
            continue

        if match:
            matches.append(match)

    unique = []
    seen = set()

    for match in matches:
        if match["uri"] not in seen:
            seen.add(match["uri"])
            unique.append(match)

    for match in unique:
        if match["label"].casefold() == label.casefold():
            return match

    normalized_label = normalize_match_label(label)

    for match in unique:
        if normalize_match_label(match["label"]) == normalized_label:
            return match

    if len(unique) == 1:
        return unique[0]

    return None


def get_lcnaf_match_by_authorized_label(label, agent_type):
    uri = get_known_lcnaf_uri_by_label(label)

    if uri:
        try:
            match = get_lcnaf_match_from_uri(uri, agent_type, "idloc_known_label")
            if match:
                return match
        except Timeout:
            raise
        except Exception:
            pass

    return get_lcnaf_search_matches(label, agent_type)


def person_label_candidates(name):
    candidates = []

    primary = clean_spaces(name.get("primary_name"))
    rest = clean_spaces(name.get("rest_of_name"))
    title = clean_spaces(name.get("title"))
    suffix = clean_spaces(name.get("suffix"))
    dates = clean_spaces(name.get("dates"))
    qualifier = clean_spaces(name.get("qualifier"))
    sort_name = clean_spaces(name.get("sort_name"))

    def add(label):
        label = clean_spaces(label)
        if label and label not in candidates:
            candidates.append(label)

    raw_candidates = []

    if primary and rest:
        raw_candidates.append(f"{primary}, {rest}")
    elif primary:
        raw_candidates.append(primary)
    elif rest:
        raw_candidates.append(rest)

    if sort_name:
        raw_candidates.append(sort_name)

    for raw in raw_candidates:
        no_title_before_dates = re.sub(
            rf",\s*(?:{OCCUPATIONAL_TITLES_RE})\s*,\s*(\d{{3,4}}.*)$",
            r", \1",
            raw,
            flags=re.I,
        )
        add(no_title_before_dates)

        no_dates = re.sub(r",\s*\d{3,4}.*$", "", raw)
        add(no_dates)

        no_title = re.sub(
            rf",\s*(?:{OCCUPATIONAL_TITLES_RE})\.?$",
            "",
            no_dates,
            flags=re.I,
        )
        add(no_title)

        add(raw)

    base = raw_candidates[0] if raw_candidates else ""

    if base and dates:
        add(f"{base}, {dates}")
    if base and suffix and dates:
        add(f"{base}, {suffix}, {dates}")
    if base and qualifier and dates:
        add(f"{base}, {qualifier}, {dates}")
    if base and title and dates:
        add(f"{base}, {title}, {dates}")
    if base and title:
        add(f"{base}, {title}")

    return candidates


def corporate_label_candidates(name):
    candidates = []

    primary = clean_spaces(name.get("primary_name"))
    sub1 = clean_spaces(name.get("subordinate_name_1"))
    sub2 = clean_spaces(name.get("subordinate_name_2"))
    number = clean_spaces(name.get("number"))
    dates = clean_spaces(name.get("dates"))
    qualifier = clean_spaces(name.get("qualifier"))
    sort_name = clean_spaces(name.get("sort_name"))

    def add(label):
        label = clean_spaces(label)
        if label and label not in candidates:
            candidates.append(label)

    dot_parts = [p for p in [primary, sub1, sub2] if p]

    if dot_parts:
        label = ". ".join(dot_parts)
        for part in [number, dates, qualifier]:
            if part:
                label = f"{label}, {part}"
        add(label)

    space_parts = [p for p in [primary, sub1, sub2] if p]

    if space_parts:
        add(" ".join(space_parts))

    if sort_name:
        add(sort_name)

    return candidates


def family_label_candidates(name):
    raw = clean_spaces(
        name.get("family_name")
        or name.get("primary_name")
        or name.get("sort_name")
    )

    if not raw:
        return []

    raw = re.sub(r"\.+$", "", raw).strip()

    if raw.lower().endswith(" family"):
        return [raw]

    return [f"{raw} family", raw]


def name_label_candidates(name, agent_type):
    if agent_type == "people":
        return person_label_candidates(name)
    if agent_type == "corporate_entities":
        return corporate_label_candidates(name)
    if agent_type == "families":
        return family_label_candidates(name)
    return []


def split_person_label(label):
    parts = [p.strip() for p in label.split(",") if p.strip()]

    parsed = {
        "primary_name": "",
        "rest_of_name": "",
        "title": "",
        "suffix": "",
        "dates": "",
        "qualifier": "",
        "name_order": "inverted",
    }

    if not parts:
        parsed["primary_name"] = label.strip()
        parsed["name_order"] = "direct"
        return parsed

    parsed["primary_name"] = parts[0]

    for part in parts[1:]:
        lower = part.lower()

        if re.search(r"\d{3,4}|active|approximately|born|died|century", lower):
            parsed["dates"] = part
        elif lower in {"jr.", "sr.", "jr", "sr", "ii", "iii", "iv", "v"}:
            parsed["suffix"] = part
        elif any(
            word in lower
            for word in [
                "sir",
                "dame",
                "saint",
                "pope",
                "king",
                "queen",
                "emperor",
                "empress",
                "duke",
                "duchess",
                "bishop",
                "archbishop",
                "cardinal",
                "rabbi",
                "imam",
            ]
        ):
            parsed["title"] = part
        elif not parsed["rest_of_name"]:
            parsed["rest_of_name"] = part
        elif not parsed["qualifier"]:
            parsed["qualifier"] = part
        else:
            parsed["qualifier"] = f"{parsed['qualifier']}, {part}"

    if not parsed["rest_of_name"]:
        parsed["name_order"] = "direct"

    return parsed


def split_corporate_label(label):
    main = label.strip()
    trailing_parts = []

    if "," in main:
        name_part, *extras = [p.strip() for p in main.split(",")]
        main = name_part
        trailing_parts = extras

    hierarchy = [p.strip() for p in main.split(". ") if p.strip()]

    parsed = {
        "primary_name": hierarchy[0] if hierarchy else label.strip(),
        "subordinate_name_1": hierarchy[1] if len(hierarchy) > 1 else "",
        "subordinate_name_2": ". ".join(hierarchy[2:]) if len(hierarchy) > 2 else "",
        "number": "",
        "dates": "",
        "qualifier": "",
    }

    for part in trailing_parts:
        if re.search(r"\d{3,4}", part):
            parsed["dates"] = part
        elif re.search(r"^\d", part):
            parsed["number"] = part
        elif not parsed["qualifier"]:
            parsed["qualifier"] = part
        else:
            parsed["qualifier"] = f"{parsed['qualifier']}, {part}"

    return parsed


def split_family_label(label):
    family_name = re.sub(r"\s+family$", "", label.strip(), flags=re.I).strip()
    return {"family_name": family_name}


def clear_parsed_name_fields(name, agent_type):
    common_fields = ["sort_name"]

    if agent_type == "people":
        fields = common_fields + [
            "primary_name",
            "rest_of_name",
            "title",
            "suffix",
            "dates",
            "qualifier",
            "name_order",
        ]
    elif agent_type == "corporate_entities":
        fields = common_fields + [
            "primary_name",
            "subordinate_name_1",
            "subordinate_name_2",
            "number",
            "dates",
            "qualifier",
        ]
    elif agent_type == "families":
        fields = common_fields + ["family_name", "prefix", "dates", "qualifier"]
    else:
        fields = common_fields

    for field in fields:
        name.pop(field, None)


def update_name_from_lc(existing_name, agent_type, match):
    updated = deepcopy(existing_name)

    clear_parsed_name_fields(updated, agent_type)

    updated["jsonmodel_type"] = AGENT_ENDPOINTS[agent_type]["name_jsonmodel_type"]
    updated["authority_id"] = match["uri"]
    updated["rules"] = "rda"
    updated["source"] = "naf"
    updated["authorized"] = True
    updated["is_display_name"] = True
    updated["sort_name_auto_generate"] = True

    if agent_type == "people":
        parsed = split_person_label(match["label"])
    elif agent_type == "corporate_entities":
        parsed = split_corporate_label(match["label"])
    elif agent_type == "families":
        parsed = split_family_label(match["label"])
    else:
        parsed = {}

    for key, value in parsed.items():
        if value:
            updated[key] = value

    return updated


def existing_name_label(name, agent_type):
    if agent_type == "corporate_entities":
        parts = [
            clean_spaces(name.get("primary_name")),
            clean_spaces(name.get("subordinate_name_1")),
            clean_spaces(name.get("subordinate_name_2")),
        ]
        return ". ".join(p for p in parts if p) or clean_spaces(name.get("sort_name"))

    if agent_type == "people":
        primary = clean_spaces(name.get("primary_name"))
        rest = clean_spaces(name.get("rest_of_name"))
        dates = clean_spaces(name.get("dates"))

        label = f"{primary}, {rest}" if primary and rest else primary or rest
        if dates:
            label = f"{label}, {dates}"
        return label or clean_spaces(name.get("sort_name"))

    if agent_type == "families":
        return clean_spaces(name.get("family_name") or name.get("sort_name"))

    return clean_spaces(name.get("sort_name"))


def make_deprecated_corporate_name(old_name):
    deprecated = deepcopy(old_name)

    deprecated["jsonmodel_type"] = "name_corporate_entity"
    deprecated["authorized"] = False
    deprecated["is_display_name"] = False
    deprecated["sort_name_auto_generate"] = True

    deprecated.pop("authority_id", None)
    deprecated.pop("sort_name", None)

    if not deprecated.get("rules"):
        deprecated["rules"] = "local"

    if not deprecated.get("source"):
        deprecated["source"] = "local"

    return deprecated


def should_preserve_deprecated_corporate_name(old_name, new_name, match):
    old_label = existing_name_label(old_name, "corporate_entities")
    new_label = existing_name_label(new_name, "corporate_entities")

    if not old_label or not new_label:
        return False

    if normalized_name_label(old_label) == normalized_name_label(new_label):
        return False

    old_normalized = normalized_name_label(old_label)

    for variant_label in match.get("variant_labels", []):
        if normalized_name_label(variant_label) == old_normalized:
            return True

    return False


def parse_conflicting_record_from_response(text):
    try:
        data = json.loads(text)
        values = data.get("error", {}).get("conflicting_record", [])
        if values:
            return values[0]
    except Exception:
        pass

    match = re.search(r"/agents/(people|corporate_entities|families)/\d+", text)
    return match.group(0) if match else None


def merge_agent_into_conflicting_record(client, source_uri, destination_uri):
    payload = {
        "uri": "merge_requests/agent",
        "jsonmodel_type": "merge_request",
        "merge_destination": {"ref": destination_uri},
        "merge_candidates": [{"ref": source_uri}],
    }

    response = client.post("merge_requests/agent", json=payload)

    if response.status_code >= 400:
        raise ValueError(
            f"ArchivesSpace rejected merge {source_uri} -> {destination_uri}: "
            f"status={response.status_code} response={response.text}"
        )

    return response.json()


def iter_agent_ids(client, agent_type):
    endpoint = AGENT_ENDPOINTS[agent_type]["path"]
    response = client.get(endpoint, params={"all_ids": True})
    response.raise_for_status()

    for agent_id in response.json():
        yield agent_id


def find_lcnaf_match_for_agent_name(name, agent_type):
    attempts = []
    timeout_errors = []

    for label in name_label_candidates(name, agent_type):
        attempts.append(label)

        try:
            match = get_lcnaf_match_by_authorized_label(label, agent_type)
        except Timeout as exc:
            timeout_errors.append(f"{label}: {exc}")
            continue
        except Exception as exc:
            return None, attempts, f"{label}: {exc}"

        if match:
            return match, attempts, None

    if timeout_errors:
        return None, attempts, {
            "timeout": True,
            "messages": timeout_errors,
        }

    return None, attempts, None


def process_agent(client, agent_type, agent_id, dry_run=True, delay=0.2):
    endpoint = AGENT_ENDPOINTS[agent_type]["path"]

    response = client.get(f"{endpoint}/{agent_id}")
    response.raise_for_status()
    agent = response.json()

    if has_id_loc_authority(agent):
        return {
            "agent_type": agent_type,
            "id": agent_id,
            "status": "skipped_existing_id_loc_authority",
            "agent_uri": agent.get("uri"),
        }

    existing_name = best_existing_name(agent)

    if not existing_name:
        return {
            "agent_type": agent_type,
            "id": agent_id,
            "status": "skipped_no_name",
            "agent_uri": agent.get("uri"),
        }

    match, attempts, error = find_lcnaf_match_for_agent_name(existing_name, agent_type)
    time.sleep(delay)

    if error:
        if isinstance(error, dict) and error.get("timeout"):
            return {
                "agent_type": agent_type,
                "id": agent_id,
                "status": "skipped_lcnaf_timeout",
                "agent_uri": agent.get("uri"),
                "attempted_labels": attempts,
                "errors": error.get("messages", []),
            }

        return {
            "agent_type": agent_type,
            "id": agent_id,
            "status": "error_lcnaf_lookup",
            "agent_uri": agent.get("uri"),
            "attempted_labels": attempts,
            "error": error,
        }

    if not match:
        return {
            "agent_type": agent_type,
            "id": agent_id,
            "status": "skipped_no_lcnaf_match",
            "agent_uri": agent.get("uri"),
            "attempted_labels": attempts,
        }

    old_names = deepcopy(agent.get("names", []))
    new_name = update_name_from_lc(existing_name, agent_type, match)

    for name in agent.get("names", []):
        name["is_display_name"] = False
        name["authorized"] = False

    deprecated_name_added = None
    names_to_insert = [new_name]

    if (
        agent_type == "corporate_entities"
        and should_preserve_deprecated_corporate_name(existing_name, new_name, match)
    ):
        deprecated_name_added = make_deprecated_corporate_name(existing_name)
        names_to_insert.append(deprecated_name_added)

    remaining_names = [
        name for name in agent.get("names", [])
        if name is not existing_name
    ]

    agent["names"] = names_to_insert + remaining_names

    report = {
        "agent_type": agent_type,
        "id": agent_id,
        "status": "dry_run" if dry_run else "updated",
        "agent_uri": agent.get("uri"),
        "match_method": match["method"],
        "attempted_labels": attempts,
        "lc_uri": match["uri"],
        "lc_label": match["label"],
        "lc_types": match["types"],
        "lc_variant_labels": match.get("variant_labels", []),
        "deprecated_name_added": deprecated_name_added,
        "old_display_name": old_names[0] if old_names else None,
        "new_display_name": new_name,
    }

    if not dry_run:
        post_response = client.post(f"{endpoint}/{agent_id}", json=agent)

        if post_response.status_code >= 400:
            conflicting_uri = parse_conflicting_record_from_response(post_response.text)

            if (
                post_response.status_code == 400
                and "Authority ID must be unique" in post_response.text
                and conflicting_uri
            ):
                source_uri = agent.get("uri")
                merge_response = merge_agent_into_conflicting_record(
                    client=client,
                    source_uri=source_uri,
                    destination_uri=conflicting_uri,
                )

                report["status"] = "merged_conflicting_authority_id"
                report["merge_source"] = source_uri
                report["merge_destination"] = conflicting_uri
                report["merge_response"] = merge_response
                return report

            raise ValueError(
                f"ArchivesSpace rejected {agent_type} agent {agent_id}: "
                f"status={post_response.status_code} "
                f"response={post_response.text}"
            )

        report["post_response"] = post_response.json()

    return report


def compact_name_for_review(name, agent_type):
    if not name:
        return ""

    if agent_type == "people":
        primary = clean_spaces(name.get("primary_name"))
        rest = clean_spaces(name.get("rest_of_name"))
        dates = clean_spaces(name.get("dates"))
        label = f"{primary}, {rest}" if primary and rest else primary or rest
        if dates:
            label = f"{label}, {dates}"
        return label or clean_spaces(name.get("sort_name"))

    if agent_type == "corporate_entities":
        parts = [
            clean_spaces(name.get("primary_name")),
            clean_spaces(name.get("subordinate_name_1")),
            clean_spaces(name.get("subordinate_name_2")),
        ]
        return ". ".join(p for p in parts if p) or clean_spaces(name.get("sort_name"))

    if agent_type == "families":
        return clean_spaces(name.get("family_name") or name.get("sort_name"))

    return clean_spaces(name.get("sort_name"))


def review_row_from_report(report):
    return {
        "apply": "no",
        "agent_type": report.get("agent_type", ""),
        "id": report.get("id", ""),
        "agent_uri": report.get("agent_uri", ""),
        "status": report.get("status", ""),
        "match_method": report.get("match_method", ""),
        "old_name": compact_name_for_review(
            report.get("old_display_name"),
            report.get("agent_type"),
        ),
        "new_name": compact_name_for_review(
            report.get("new_display_name"),
            report.get("agent_type"),
        ),
        "lc_label": report.get("lc_label", ""),
        "lc_uri": report.get("lc_uri", ""),
        "deprecated_name_added": compact_name_for_review(
            report.get("deprecated_name_added"),
            report.get("agent_type"),
        ),
        "lc_variant_labels": " | ".join(report.get("lc_variant_labels", [])),
        "attempted_labels": " | ".join(report.get("attempted_labels", [])),
    }


def apply_review_csv(client, csv_path, delay=0.2):
    counts = {}

    with open(csv_path, "r", encoding="utf-8-sig", newline="") as f:
        reader = csv.DictReader(f)

        for row in reader:
            decision = clean_spaces(row.get("apply")).casefold()

            if decision not in {"yes", "y", "true", "1", "update"}:
                continue

            agent_type = clean_spaces(row.get("agent_type"))
            agent_id = int(row["id"])

            try:
                report = process_agent(
                    client=client,
                    agent_type=agent_type,
                    agent_id=agent_id,
                    dry_run=False,
                    delay=delay,
                )
            except Exception as exc:
                report = {
                    "agent_type": agent_type,
                    "id": agent_id,
                    "status": "error",
                    "error": str(exc),
                }

            counts[report["status"]] = counts.get(report["status"], 0) + 1
            print(json.dumps(report, ensure_ascii=False))

    print(json.dumps({"summary": counts}, indent=2))


def main():
    parser = argparse.ArgumentParser(
        description="Reconcile ArchivesSpace agent records to LCNAF using id.loc.gov linked data."
    )
    parser.add_argument("--secrets", default="secrets.json")
    parser.add_argument(
        "--agent-type",
        choices=AGENT_ENDPOINTS.keys(),
        default=None,
        help="Process only one agent type.",
    )
    parser.add_argument(
        "--apply",
        action="store_true",
        help="Actually update ArchivesSpace. Default is dry run.",
    )
    parser.add_argument(
        "--apply-review-csv",
        default=None,
        help="Apply only rows marked apply=yes in this CSV.",
    )
    parser.add_argument("--limit", type=int, default=None)
    parser.add_argument("--delay", type=float, default=0.2)
    parser.add_argument(
        "--output",
        default="lcnaf_agent_updates.jsonl",
        help="JSONL report file.",
    )
    parser.add_argument(
        "--review-csv",
        default=None,
        help="Write a CSV review file during dry run.",
    )

    args = parser.parse_args()

    dry_run = not args.apply
    client = make_client(load_secrets(args.secrets))

    if args.apply_review_csv:
        apply_review_csv(client, args.apply_review_csv, delay=args.delay)
        return

    agent_types = [args.agent_type] if args.agent_type else list(AGENT_ENDPOINTS)
    counts = {}

    review_file = None
    review_writer = None

    if args.review_csv:
        review_file = open(args.review_csv, "w", encoding="utf-8", newline="")
        fieldnames = [
            "apply",
            "agent_type",
            "id",
            "agent_uri",
            "status",
            "match_method",
            "old_name",
            "new_name",
            "lc_label",
            "lc_uri",
            "deprecated_name_added",
            "lc_variant_labels",
            "attempted_labels",
        ]
        review_writer = csv.DictWriter(review_file, fieldnames=fieldnames)
        review_writer.writeheader()

    try:
        with open(args.output, "w", encoding="utf-8") as out:
            for agent_type in agent_types:
                for index, agent_id in enumerate(iter_agent_ids(client, agent_type), start=1):
                    if args.limit and index > args.limit:
                        break

                    try:
                        report = process_agent(
                            client=client,
                            agent_type=agent_type,
                            agent_id=agent_id,
                            dry_run=dry_run,
                            delay=args.delay,
                        )
                    except Exception as exc:
                        report = {
                            "agent_type": agent_type,
                            "id": agent_id,
                            "status": "error",
                            "error": str(exc),
                        }

                    counts[report["status"]] = counts.get(report["status"], 0) + 1

                    line = json.dumps(report, ensure_ascii=False)
                    print(line)
                    out.write(line + "\n")
                    out.flush()

                    if review_writer and report.get("status") == "dry_run":
                        review_writer.writerow(review_row_from_report(report))
                        review_file.flush()

    finally:
        if review_file:
            review_file.close()

    print(json.dumps({"summary": counts}, indent=2))


if __name__ == "__main__":
    main()
