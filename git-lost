#!/usr/bin/env python3
import json
import os.path
import subprocess
from typing import List

import click
import git

CONFIG_PATH = os.getenv('HOME') + '/.config/git-lost.json'

DEFAULT_CONFIG = {
    # names of branches that should never be cleaned up
    'permanent_branches': ['master', 'main'],
    # whether local refs should be created for each permanent branch
    'always_create_permanent_branches': True,
    # whether to do a git fetch first
    'fetch_first': True,
    # whether to use --tags option when fetching
    'fetch_tags': True,
    # whether to use --prune option when fetching
    'fetch_prune': True,
    # whether local branches should be automatically fast-forwarded.
    # Possible values are:
    #   "ff_permanent" -> Only permanent branches will be fast-forwarded
    #   "ff_all"       -> All locally checked out branches will be
    #                     fast-forwarded. NOTE: this only works if the branch
    #                     names match *and* the local branch's upstream is
    #                     configured correctly.
    #   "ff_none"      -> Only permanent branches will be fast-forwarded
    # FIXME: make the default ff_permanent once the FF feature is working
    'auto_fast_forward': 'ff_none',
    # which remote repos to compare with
    'upstream_names': ['origin'],
}


# read/write config
def _get_config():
    if not os.path.exists(CONFIG_PATH):
        return DEFAULT_CONFIG
    with open(CONFIG_PATH) as f:
        # FIXME: it'd be nice to validate the config to make sure it's not
        # going to break anything
        data = json.load(f)

    ret = DEFAULT_CONFIG.copy()
    ret.update(data)
    return ret


def _run(cmd):
    return subprocess.check_output(cmd).decode('utf-8')


def _get_refs():
    local = set()
    remote = {}
    tags = set()

    # get a list of local branches, remote branches, and tags
    for line in _run(['git', 'show-ref']).splitlines():
        sha, ref = line.split(' ')
        assert ref.startswith('refs/'), ref
        if ref.startswith('refs/heads/'):
            local.add(ref[11:])
        elif ref.startswith('refs/tags/'):
            tags.add(ref[10:])
        elif ref.startswith('refs/remotes/'):
            if not ref.endswith('/HEAD'):
                name, branch = ref[13:].split('/', 1)
                remote.setdefault(name, set()).add(branch)
        elif ref == 'refs/stash':
            continue
        else:
            raise Exception("Unexpected ref %r" % ref)

    return local, remote, tags


def _do_fast_forward(repo, name, remote, mustExist=True):
    raise Exception("TODO: not complete")  # noqa


def _createPermanent(repo: git.Repo, name: str, remotes: List[str]) -> None:
    def _err(msg):
        click.secho("WARNING: Can't create permanent branch {}: {}".format(name, msg),
                    fg='red')

    click.secho('Creating permanent branch {}'.format(name), fg='green', dim=True)

    if not remotes:
        _err("No remotes configured")
        return

    # work out which remote(s) have the branch we want
    candidateRemotes = [
        r for r in repo.remotes
        if r.name in remotes
        and any(ref.name == '{}/{}'.format(r.name, name) for ref in r.refs)
    ]

    if not len(candidateRemotes):
        _err("No remotes configured, or no remotes have branch {}".format(name))
        return

    if len(candidateRemotes) > 1:
        _err("Multiple remotes have a {} branch".format(name))
        return

    remote = candidateRemotes[0]

    # create the branch
    repo.git.branch('--track', name, '{}/{}'.format(remote.name, name))


def _examine_branch(repo, head, remotes, permanent):
    # get a list of branches containing this one
    lines = _run(['git', 'branch', '--all', '--contains=' + head.name])
    descendants = set(line[2:].rstrip() for line in lines.splitlines())

    # remove the branch from its list of descendants
    descendants.remove(head.name)

    for perm in permanent:
        # if the branch is merged to one of the permanent branches locally, we
        # should remove it
        if perm in descendants:
            _request_removal(repo, head, perm, permanent[0])
            return

        # if the branch is merged to one of the remote permanent branches, we
        # should remove it
        for remote in remotes:
            fullname = 'remotes/{}/{}'.format(remote, perm)
            if fullname in descendants:
                _request_removal(repo, head, fullname, permanent[0])
                return

    # check remote branches first
    try_again = set()
    for other in descendants:
        if other.startswith('remotes/'):
            if _request_removal(repo, head, other, permanent[0]):
                return
        else:
            try_again.add(other)

    for other in try_again:
        if _request_removal(repo, head, other, permanent[0]):
            return


def _attempt_removal(repo, local, remotename, other):
    if remotename is None:
        otherref = other
    else:
        otherref = "/".join((remotename, other))

    current = None if repo.head.is_detached else repo.active_branch.name

    # don't remove the current branch if it's only pushed upstream
    if local == current and local == other:
        return False

    # is it an ancestor?
    merge_base = _run(['git', 'merge-base', local, otherref]).strip()
    sha = _run(['git', 'rev-parse', '--verify', local]).strip()
    if sha == merge_base:
        msg = "{} is merged to {}. Delete it?".format(local, otherref)

        if click.confirm(msg, default=False):
            # do we need to jump to another branch first?
            if current == local:
                # jump to the other thing first
                _run(['git', 'checkout', otherref])

            click.echo("Deleting local branch {} (was at {})."
                       .format(local, sha))
            _run(['git', 'branch', '-D', local])
        return True
    return False


def _request_removal(repo, head, othername, defaultJump):
    current = None if repo.head.is_detached else repo.active_branch.name

    msg = "{} is merged to {}. Delete it?".format(
        click.style(head.name, fg='white', bold=True),
        click.style(othername, fg='white', bold=False))

    if click.confirm(msg, default=False):
        # do we need to jump to another branch first?
        if current == head.name:
            # jump to one of the permanent branches first
            _run(['git', 'checkout', defaultJump])

        sha = head.commit.hexsha

        msg = (click.style('Deleting local branch ', fg='red')
               + click.style(head.name, fg='red', bold=True)  # noqa: W503
               + click.style(' (was at ', fg='red')           # noqa: W503
               + click.style(sha[:8], fg='red', bold=True)    # noqa: W503
               + click.style(').', fg='red')                  # noqa: W503
               )
        click.echo(msg)
        _run(['git', 'branch', '-D', head.name])
        return True
    return False


@click.command()
@click.option('-f/-F', '--fetch/--no-fetch', is_flag=True, default=None)
@click.option('-p', '--permanent', multiple=True)
@click.argument('remote', nargs=-1)
def main(fetch, permanent, remote):
    # get branch list
    repo = git.Repo()
    local, remote, _ = _get_refs()

    conf = _get_config()

    # what are the permanent branches we don't want to destroy?
    if len(permanent):
        permanent = list(permanent)
    else:
        permanent = conf['permanent_branches']

    # which remotes are we working with?
    if len(remote):
        remotes = list(remote)
    else:
        remotes = conf['upstream_names']

    if not len(remotes):
        raise click.ClickException('No remotes nominated')

    if fetch is None:
        fetch = conf.get('fetch_first', True)

    # 1) get fetch if needed
    if fetch:
        click.secho('Fetching from {} ...'.format(', '.join(remotes)),
                    fg='cyan')
        cmd = ['git', 'fetch', '--multiple']
        if conf['fetch_prune']:
            cmd.append('--prune')
        if conf['fetch_tags']:
            cmd.append('--tags')
        cmd.extend(remotes)
        _run(cmd)

    # 2) create permanent branches if needed
    if conf['always_create_permanent_branches']:
        for name in permanent:
            if not hasattr(repo.heads, name):
                _createPermanent(repo, name, remotes)

    # 3) auto-fast forward any branches that need fastforwarding
    if conf['auto_fast_forward'] in ('ff_permanent', 'ff_all'):
        if len(remotes) == 1:
            for name in permanent:
                _do_fast_forward(repo, name, remotes[0], False)

    if conf['auto_fast_forward'] == 'ff_all':
        raise Exception("TODO: auto-fast-forward other branches")  # noqa

    # iterate through local branches and try to delete them one by one
    for head in repo.heads:
        # don't attempt to clean up permanent branches
        if head.name in permanent:
            click.secho('Not cleaning up permanent branch ',
                        fg='blue', nl=False)
            click.secho(head.name, fg='blue', bold=True)
            continue

        # find out if this thing is merged anywhere?
        _examine_branch(repo, head, remotes, permanent)


if __name__ == '__main__':
    main()
