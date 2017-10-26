FROM python:2.7-alpine
EXPOSE 5000
WORKDIR /usr/src/app
ENTRYPOINT ["dumb-init", "-r", "15:2"]
# Before attempting to replace this command, please read up on https://github.com/Yelp/dumb-init .
CMD ["sh", "-c", "exec ./wait-for ${POGOMAP_DB_HOST}:3306 -- python ./runserver.py --host 0.0.0.0 -cf /mnt/config.ini -l \"${RM_LOC}\" -sn \"${RM_NAME}\" -ac \"${RM_ACCOUNTS}\" ${RM_MODE}"]
# At run-time, override these environment variables.
# RM_MODE should be either "-ns" or "-os"; you need one -os for a graphical map,
#     and several -ns for the areas that need scanning.
# RM_ARGS is whatever extra arguments you need.
ENV RM_ACCOUNTS=/mnt/accounts.csv RM_LOC="99.99, 99.99" RM_NAME="City Hex" RM_MODE="-ns" RM_ARGS=""

# You should aim to put most of your config into a config file shared by all RM
# containers. Use a single-file volume to replace this
# (ie, `docker run -v /path/to/config.ini:/mnt/config.ini`).
COPY config.ini.example /mnt/config.ini

RUN apk add --no-cache ca-certificates openssl

RUN wget https://raw.githubusercontent.com/Eficode/wait-for/master/wait-for && chmod +x wait-for
RUN wget https://raw.githubusercontent.com/RocketMap/RocketMap/develop/requirements.txt
RUN apk add --no-cache build-base git libffi libffi-dev openssl-dev python-dev linux-headers \
 && pip install --no-cache-dir dumb-init \
 && pip install --no-cache-dir -r requirements.txt \
 && apk del --no-cache -v build-base git libffi-dev openssl-dev python-dev linux-headers

RUN wget https://github.com/RocketMap/RocketMap/archive/develop.zip \
 && unzip develop.zip \
 && mv -fn RocketMap-develop/* . \
 && rm -r RocketMap-develop \
 && rm develop.zip

RUN apk add --no-cache nodejs \
 && npm install \
 && npm run build \
 && rm -r node_modules \
 && apk del nodejs
