#
# Copyright (c) 2008-2018 the Urho3D project.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.
#

task default: %w[build]

# Usage: NOT intended to be used manually
desc 'Build and install Clang as prebuilt binaries for Travis CI'
task :build do
  next unless ENV['BRANCH']
  next if timeup
  system 'mkdir -p src build && cd src && git clone -b $BRANCH https://github.com/llvm-mirror/llvm.git && cd llvm/tools && git clone -b release_60 https://github.com/llvm-mirror/clang.git && cd clang/tools && git clone -b release_60 https://github.com/llvm-mirror/clang-tools-extra.git extra && cd ../../../projects && git clone -b release_60 https://github.com/llvm-mirror/compiler-rt.git && cd ../../../build && cmake -DCMAKE_BUILD_TYPE=Release -DLLVM_TOOL_CLANG_TOOLS_EXTRA_BUILD=1 -DLLVM_TARGETS_TO_BUILD=X86 -G "Unix Makefiles" ../src/llvm' or abort 'Failed to clone LLVM/Clang'
  if !wait_for_block { Thread.current[:subcommand_to_kill] = 'make'; system 'cd build && DESTDIR=~ cmake --build . -- -j$NUMJOBS install' }
    abort 'Failed to build and install LLVM/Clang' unless File.exists?('already_timeup.log')
    $stderr.puts "Skipped the rest of the CI processes due to insufficient time"
    next
  end
  system "cd ~/usr && git clone --depth 1 -q -b $BRANCH https://github.com/urho3d/llvm-clang.git && rsync -a --delete --exclude .git local/ llvm-clang && cd llvm-clang && git config user.name '#{ENV['GIT_NAME']}' && git config user.email '#{ENV['GIT_EMAIL']}' && git remote set-url --push origin https://#{ENV['GH_TOKEN']}@github.com/urho3d/llvm-clang.git && git add -Af . >/dev/null 2>&1 && (git commit -q -m \"Travis CI: LLVM/Clang installation at #{Time.now.utc}.\" || true) && git push -q -u origin $BRANCH >/dev/null 2>&1" or abort 'Failed to push the llvm-clang'
end

# Usage: NOT intended to be used manually
desc 'Check whether the build job should be retriggered again'
task :continuation do
  ours = `git log -1 --pretty=format:%ct`
  theirs = `git fetch origin $BRANCH && git log -1 --pretty=format:%ct FETCH_HEAD`
  if ours > theirs
    system 'travis login -g $GH_TOKEN --org --no-interactive >/dev/null 2>&1 && travis restart --org --no-interactive -r $TRAVIS_REPO_SLUG >/dev/null 2>&1' or abort 'Failed to retrigger the job'
  end
end

# Always call this function last in the multiple conditional check so that the checkpoint message does not being echoed unnecessarily
def timeup quiet = false, cutoff_time = 40.0
  unless File.exists?('start_time.log')
    system 'touch start_time.log split_time.log'
    return nil
  end
  current_time = Time.now
  elapsed_time = (current_time - File.atime('start_time.log')) / 60.0
  unless quiet
    lap_time = (current_time - File.atime('split_time.log')) / 60.0
    system 'touch split_time.log'
    puts "\n=== elapsed time: #{elapsed_time.to_i} minutes #{((elapsed_time - elapsed_time.to_i) * 60.0).round} seconds, lap time: #{lap_time.to_i} minutes #{((lap_time - lap_time.to_i) * 60.0).round} seconds ===\n\n" unless File.exists?('already_timeup.log'); $stdout.flush
  end
  return system('touch already_timeup.log') if elapsed_time > cutoff_time
end

# Usage: wait_for_block('This is a long function call...') { call_a_func } or abort
#        wait_for_block('This is a long system call...') { system 'do_something' } or abort
def wait_for_block comment = '', retries = -1, retry_interval = 60
  # Wait until the code block is completed or when it exceeds the number of retries (if the retries parameter is provided)
  thread = Thread.new { rc = yield; Thread.main.wakeup; rc }
  thread.priority = 1   # Make the worker thread has higher priority than the main thread
  str = comment
  retries = retries * 60 / retry_interval unless retries == -1
  until thread.status == false
    if retries == 0 || timeup(true)
      thread.kill
      # Also kill the child subproceses spawned by the worker thread if specified
      system "killall #{thread[:subcommand_to_kill]}" if thread[:subcommand_to_kill]
      sleep 5
      break
    end
    print str; str = '.'; $stdout.flush   # Flush the standard output stream in case it is buffered to prevent Travis-CI into thinking that the build/test has stalled
    retries -= 1 if retries > 0
    sleep retry_interval
  end
  puts "\n" if str == '.'; $stdout.flush
  thread.join
  return thread.value
end

# vi: set ts=2 sw=2 expandtab:
