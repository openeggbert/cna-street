# Patches contributed to CNA

Changes this project needed that belonged in the framework rather than in the
application, kept here as `git format-patch` output so the repository carries
its own record of them: a reader of cna-street can see exactly what it asked of
CNA without having to go and find a branch.

The rule for what lands here rather than being worked around in the
application: a gap is worked around when it is this project's problem, and
fixed upstream when it is the framework's. Most of `../cna-findings.md` is the
first kind.

| Patch | Upstream | Finding |
| --- | --- | --- |
| `0001-feat-cnb-generate-mip-chains-when-compiling-a-source.patch` | `openeggbert/cna`, branch `feat/cnb-source-mipmaps`, commit `347139500` | [CNA-F12](../cna-findings.md) |

## 0001 — mip chains in the content pipeline

`cna_tool_source_to_cnb` compiled every texture with exactly one mip level. The
container had always been able to carry a chain — `TEXH` declares a mip count,
`TEXD` carries one payload per level, and the `Texture2D` loader uploads every
level it finds — but nothing produced one.

The consequence is not subtle: a texture with one level aliases on anything seen
at a grazing angle, which is most of a street, so compiling this project's
textures made it look *worse* than generating them at run time. No
application-side workaround recovers that without throwing away most of what the
pipeline is for.

The patch adds `CNA::Content::Cnb::GenerateRgba8MipChain` and a `--mipmaps`
option on the compiler, with the averaging colour space as an argument rather
than an assumption: averaging sRGB-encoded texels as if they were light darkens
every level, while normals, roughness and masks must be averaged exactly as they
are stored. Five GTest cases ship with it in CNA's own suite;
`../../tests/ContentPipelineTests.cpp` exercises the same function here.

### Applying it

The branch is pushed, so the ordinary route is to fetch it:

```sh
git -C ../cna fetch origin feat/cnb-source-mipmaps
git -C ../cna checkout feat/cnb-source-mipmaps
```

Or apply the patch to any base:

```sh
git -C ../cna am ../cna-street/docs/patches/0001-*.patch
```

Without it, cna-street still runs — the content pipeline is optional and every
surface that is not compiled is generated at start-up instead. What you lose is
the ability to *use* the compiled set without the street aliasing.
