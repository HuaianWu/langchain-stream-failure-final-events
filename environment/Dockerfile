FROM public.ecr.aws/x8v8d7g8/mars-base:latest
WORKDIR /app

# Git time-travel: clone, then make the repo's default branch point AT the base
# commit with no future history — a real branch checkout (not a detached HEAD),
# future commits/tags gc'd away so the reference solution can't leak from history.
ARG BASE_SHA=49da5817a8c518357cf188fc05f7579bbe58986a
RUN git clone https://github.com/langchain-ai/langchain . \
 && DEFAULT="$(git remote show origin | sed -n 's/.*HEAD branch: //p')" \
 && git checkout -B "$DEFAULT" "$BASE_SHA" \
 && git remote remove origin \
 && for b in $(git for-each-ref --format='%(refname:short)' refs/heads | grep -vx "$DEFAULT"); do git branch -D "$b" || true; done \
 && for t in $(git tag); do git merge-base --is-ancestor "$t" HEAD 2>/dev/null || git tag -d "$t"; done \
 && git reflog expire --expire=now --all \
 && git gc --prune=now \
 && { test ! -s .gitmodules || git submodule update --init --recursive; }
# langchain-core is its own uv project (libs/core/uv.lock with a `test` dependency group, which is what
# upstream CI syncs). mars-base's uv 0.9.18 cannot read this lockfile; the environment lives outside /app.
ENV UV_PROJECT_ENVIRONMENT=/opt/venv
RUN pip install --no-cache-dir "uv==0.12.9" \
    && cd libs/core \
    && python -m uv sync --frozen --no-default-groups --group test \
    && test -z "$(git status --porcelain)"
ENV PATH="/opt/venv/bin:${PATH}"

# Disable git commit hooks (pre-commit, husky etc.): dev-workflow tooling, not task content.
# Broken hook environments otherwise block the agent's (and oracle's) commits.
RUN cd /app && git config core.hooksPath /dev/null

CMD ["/bin/bash"]
