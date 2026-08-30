FROM node:16-alpine AS builder

WORKDIR /frontend

COPY package.json package-lock.json /frontend/

RUN npm ci

COPY . .

RUN npm run build

