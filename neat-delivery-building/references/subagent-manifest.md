# Subagent Manifest

The JSON the coordinator passes in each subagent's prompt (coordinator Phase 2 Step 1). It carries paths, not file contents — the subagent reads the files itself in its own context window.

```jsonc
{
  "skill_dir": "/abs/path/neat-delivery-building",   // resolved real path of the building skill directory
  "wave": "wave01-booking",
  "wave_file": "specs/delivery/wave01-booking.md",   // requirements, features, and tasks for the wave

  "features": [                      // 1 feature, or up to 3 small/overlapping features built sequentially
    {
      "name": "auth",
      "depends_on": [],              // done features this one depends on — their wave_file sections are context only
      "resume_from": null            // task slug to start at; null = first task
    }
  ],

  "solution_path": "specs/discovery/solution.md",
  "claude_md": "CLAUDE.md",             // null if absent
  "legacy_path": null,                  // from modernization.md frontmatter; null if not on modernization track

  "knowledge": {
    "file": "specs/system/system.md",
    "modules": [                        // modules the batch's task files map to, plus shared-modules.md
      "specs/system/modules/auth.md",
      "specs/system/modules/shared-modules.md"
    ]
  },

  "rules": [                            // building rules the subagent follows, under skill_dir
    "{skill_dir}/SKILL.md",
    "{skill_dir}/../references/neat-project/delivery-rules.md",
    "{skill_dir}/../references/neat-project/mutation-adequacy.md",
    "{skill_dir}/references/quality-gates.md",
    "{skill_dir}/references/structure-checks.md",
    "{skill_dir}/references/execution-protocol.md"
  ],

  "pre_implementation": {               // same shape as progress.json wave.pre_implementation
    "decisions_reviewed": "2026-11-07 14:00",
    "callouts": [{ "callout": "{open: auth-provider}", "resolution": "AWS Cognito" }],
    "values": { "AWS_REGION": "us-east-1" }
  },

  "decision_log": [                     // answers to earlier blocked questions, all features
    { "feature": "auth", "question": "Extend BaseService or implement IUserService?", "answer": "Implement IUserService" }
  ]
}
```

**Before spawning, the coordinator checks:**

- Features in one manifest have no `depends_on` edges between them.
- Every `depends_on` feature is `done`.
- `wave_file` and every path in `knowledge` and `rules` exist.

Compare each manifest against this schema — confirm all required keys are present.
