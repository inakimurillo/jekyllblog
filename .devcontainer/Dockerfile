FROM ubuntu:22.04

# Install dependencies
RUN apt-get update && \
    apt-get install -y curl gnupg build-essential ruby-full nodejs && \
    rm -rf /var/lib/apt/lists/*

ENV GEM_HOME="$HOME/gems" \
    PATH="$HOME/gems/bin:$PATH"

# Install Bundler
RUN gem install jekyll bundler

# Add user 'dev' with password 'dev' and add to sudo group
RUN useradd -m -s /bin/bash dev \
    && echo 'dev:dev' | chpasswd \
    && adduser dev sudo
    
# Install dependencies
RUN apt-get update && \
    apt-get install -y sudo git

# Set working directory
USER dev
WORKDIR /home/dev

# Add GEM_HOME and PATH to dev user's .bashrc
RUN echo 'export GEM_HOME="$HOME/gems"' >> /home/dev/.bashrc \
    && echo 'export PATH="$HOME/gems/bin:$PATH"' >> /home/dev/.bashrc

# Expose default Jekyll port
EXPOSE 4000

CMD ["bash"]
